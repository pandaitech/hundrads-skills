---
name: meta-ads-autopilot
description: Autonomous Meta-ads marketer loop. Fired on a schedule (durable cron), it runs ONE full unattended pass — refresh insights, classify each ad, decide scale/kill/new-test, draft creative in the account voice, submit meta_ad drafts + scale/kill proposals to the Hundrads review queue, and report the pass via Hundrads (POST /v1/reports/ads → the channel set in the dashboard Reports tab). No human in the loop mid-pass; every decision resolves from config defaults. Replaces the freelance ad manager. Use when a cron prompt says "run the meta-ads-autopilot" / "one autopilot pass", or the user asks to run/test the autopilot manually.
---

# Meta Ads Autopilot

A scheduled, self-deciding wrapper around the manual [`meta-ads`](../meta-ads/SKILL.md)
capability. Each firing = **one full pass** that runs to completion with **no
human in the loop**: analyze → decide → generate creative → submit drafts →
alert. The user is not available to answer during a pass — never stop for input.

State + tunables live in `.claude/autopilot/config.json` and
`.claude/autopilot/state.json`. Read both at the start, write `state.json` at the
end. All paths are relative to the repo root.

## API access

Everything goes through the Hundrads REST API (`/v1`) via curl. Canonical setup
(run once per pass, from the repo root):

```bash
export $(grep -E '^HUNDRADS_API_KEY=' .env | xargs)
curl -s "https://hundrads.com/v1/..." -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

POSTs add: `-X POST -H 'Content-Type: application/json' -d '{...}'`. All
responses are JSON — pipe through `python3 -m json.tool` or `jq` as needed.
Ad accounts: `GET /v1/accounts`. Brand policies: `GET /v1/policies?brand=`.

## Guardrails — read before acting (non-negotiable)

- **Money.** ONLY drafts and proposals, only via `POST /v1/drafts`. The autopilot
  never touches Meta: a `meta_ad` draft creates zero Meta objects until the user
  Approves it in the dashboard — and even then the approve handler only ever
  creates PAUSED objects. Never activate an ad, never scale a *live* ad
  yourself, never raise a live budget yourself, never delete/kill a *live* ad
  yourself. Scale and kill are *proposed* (`budget_change` / `status_change`
  drafts) and wait in the same review queue; budget increases and resume are
  NEVER auto-approvable.
- **Learning phase.** Never base a scale OR kill decision on an ad set younger
  than `guards.min_adset_age_hours` or with fewer than
  `guards.min_results_for_judgement` results. Classify those `learning` and leave
  them alone.
- **Anti-churn.** Never restage/scale an ad set you touched within
  `guards.restage_cooldown_hours` (check `state.brands[brand].staged`). Submit at
  most `guards.max_new_tests_per_brand_per_day` new test per brand per day,
  regardless of how often the loop fires (check `state.brands[brand].last_test_date`).
- **No pile-up.** Before submitting anything, run BOTH checks:
  1. `GET /v1/drafts?status=pending&brand=<brand>` — any pending drafts mean the
     user hasn't acted on the last batch.
  2. `state.brands[brand].staged` — approved-but-unactivated ads (fetch each
     `effective_status` via `GET /v1/ads?brand=`; ACTIVE or gone = cleared).
  If either shows a backlog, **do NOT submit more** — HOLD and put an "act on
  the pending X" reminder in the digest. Resume once the queue is cleared.
- **Cost.** A "nothing to do" pass makes ZERO model/poster calls. Only draft
  copy (`POST /v1/complete`) or a poster (`POST /v1/media/poster`) when actually
  submitting a new test/scale this pass. Stop generating posters once
  `poster_spend_today` would exceed `guards.max_poster_cost_usd_per_day`.
  Uploading a ready-made asset (`POST /v1/media/upload`) costs nothing and does
  not count against the poster cap — prefer it once the cap is near.
- **Quiet.** Default to silence. Discord only on a true fire (immediate) or when
  the digest tick is due (see step 5). Most passes send nothing.

**Why you may decide without asking:** the draft wall. The worst outcome of any
autonomous call is a pending draft in the Hundrads review queue the user never
approves — zero Meta objects created, zero spend, fully reversible with one
Reject click. Nothing the autopilot does can reach Meta; only the user's Approve
can, and even that only ever stages PAUSED. Keep that wall intact and you cannot
cause harm.

## One pass

### 1. Load state
Read `config.json` + `state.json`. Compute `now` (UTC). Reset
`poster_spend_today` to 0 if `poster_spend_date` != today.

### 2. Refresh data (cost-aware)
- **Daily full refresh:** if this is the first pass on/after `daily_summary_hour`
  today (compare `last_daily_summary_date`), refresh the library for each brand
  in `config.brands`:
  ```bash
  curl -s -X POST "https://hundrads.com/v1/library/refresh" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
    -d '{"target":"ads","brand":"<brand>"}'
  ```
- **Intraday passes:** skip the full refresh. Use live insights —
  `GET /v1/insights?brand=<brand>&date_preset=today&level=ad` (and
  `date_preset=yesterday` when judging) — for fire-detection only; lighter on
  the Meta API.

### 3. Classify (per brand, per active ad)
Pull ranked ads from the library:
`GET /v1/library/ads?brand=<brand>&sort=roas&active_only=true` and
`...&sort=ctr&active_only=true` (each returns `{items, count}`). Cross-reference
live insights (`GET /v1/insights`) and live status (`GET /v1/ads?brand=<brand>`).
Label each active ad:
- `rejected` — `effective_status` disapproved/with_issues → **fire**.
- `zero-delivery` — has budget but ~0 impressions over the window → **fire**.
- `spend-runaway` — daily spend > adset budget × `guards.spend_runaway_mult` → **fire**.
- `roas-crash` — ROAS dropped > `guards.roas_crash_pct` vs the ad's prior window → **fire**.
- `learning` — younger than `min_adset_age_hours` or < `min_results_for_judgement` results → leave alone.
- `winner` — out of learning, ROAS clearly above brand average, healthy CTR.
- `loser` / `fatigued` — out of learning, ROAS below average or CTR decayed.

### 4. Decide + act (autonomous, all defaults from `config.defaults`)
For each brand, in priority order, respecting every guardrail:

- **Winner with no scale staged within `restage_cooldown_hours`** → submit a
  higher-budget duplicate as a `meta_ad` draft. Budget = winner's daily budget ×
  `defaults.scale_budget_mult`. Reuse the winner's copy + image (see
  "Submitting drafts" below). Alternatively (or additionally, when raising the
  existing ad set is the better move) file a `budget_change` proposal:
  ```bash
  curl -s -X POST "https://hundrads.com/v1/drafts" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
    -d '{"kind":"budget_change","agent_note":"<pitch>","payload":{
      "brand":"<brand>","adset_id":"<id>","adset_name":"<name>",
      "current_daily_budget_cents":1500,"new_daily_budget_cents":2250,
      "reason":"<why>","expected_impact":"<what you expect>"}}'
  ```
  Mention the scale in the digest either way. Budget increases never
  auto-approve — they always wait for the user.
- **New test due** (under `max_new_tests_per_brand_per_day`, and a winner/angle
  worth iterating exists) → draft `defaults.variants_per_test` variations in the
  account voice, changing ONE variable across them. Copy comes from
  `POST /v1/complete` `{prompt, system?, provider, model, n?, temperature?,
  skill:"meta-ads-autopilot", channel:"ads", brand}` → `{results:[...],
  cost_usd}`; pick the model from the registry (`GET /v1/models`) — never from
  memory. Follow `meta-ads` voice + the organic-visuals rule:
  - **Ready-made asset available** → upload it instead of generating. When the
    brand has creative files configured in `config.creative_assets_by_brand[brand]`
    (e.g. app screenshots, product photos, designed posters), upload one:
    `curl -F brand=<brand> -F file=@<path>` to `POST /v1/media/upload` →
    `{image_hash, image_url}`. No AI call, so it costs NO poster budget and never
    touches the poster cap — prefer it when an on-brand asset fits the angle, and
    especially when `poster_spend_today` is near `guards.max_poster_cost_usd_per_day`.
  - Scene-led visual, no asset → `POST /v1/media/poster` `{prompt, complexity:"simple",
    aspect, brand, quality}` → `{image_hash, image_url}` (respect the poster
    cost cap); use the returned `image_hash`.
  - Punchline is text-inside-a-screenshot → do NOT AI-render it. If a real
    screenshot is configured in `config.creative_assets_by_brand[brand]`, upload
    it (above) and attach it. Otherwise write the exact copy into the draft, add a
    "screenshot on your real phone" note to the digest, and submit without that image.
  - Submit each variant as its own `meta_ad` draft.
- **Loser / fatigued / fire** → DO NOT act on Meta. File a pause proposal:
  ```bash
  curl -s -X POST "https://hundrads.com/v1/drafts" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
    -d '{"kind":"status_change","agent_note":"<pitch>","payload":{
      "brand":"<brand>","object_type":"adset","object_id":"<id>",
      "object_name":"<name>","action":"pause","reason":"<why>"}}'
  ```
  Flag it in the digest too. For a fatigued winner, optionally draft + submit
  ONE replacement test (counts against the daily test cap).

**Submitting drafts (campaign reuse, get-or-create by name):**

```bash
curl -s -X POST "https://hundrads.com/v1/drafts" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
  -d '{"kind":"meta_ad","agent_note":"<one-paragraph pitch: what this is and why>","payload":{
    "brand":"<brand>","campaign_name":"<campaign>","ad_name":"<name>",
    "objective":"<objective>","primary_text":"...","headline":"...",
    "description":"...","call_to_action":"<CTA>","link_url":"https://...",
    "daily_budget_cents":1500,"image_hash":"<hash>"}}'
```

Returns `{draft: {id}, warnings, review_url}`. Give every variant in the same
test the SAME `campaign_name` — on approval the handler get-or-creates the
campaign by name, so all variants land in one campaign (consolidated learning).
Resolve `objective` from the brand's top past ads (`GET /v1/library/ads` items
include `objective`); fall back to `defaults.objective_by_brand[brand]`. Use
`defaults.default_cta` and `defaults.daily_budget_cents` unless scaling. Surface
any `warnings` from the response in the digest; record each `draft.id` in
`state.brands[brand].staged`. The draft itself never touches Meta — only the
user's Approve does, and it only ever creates PAUSED objects.

**Preview each new test ad (`POST /v1/reports/ad-preview`):** right after a
`meta_ad` draft for a NEW test is accepted, send ONE preview message per ad so
the user sees what it looks like before approving — a rendered Facebook/Instagram
**feed placement mockup** (page avatar → primary text → creative → headline /
description / CTA button) plus the copy + budget. Same delivery as the digest:
the server resolves the channel + webhook from the dashboard **Reports** tab and
renders the image itself; the autopilot never holds a webhook. Pass the SAME
creative fields you put on the draft, plus the `image_url` you got back from
`POST /v1/media/poster` or `/v1/media/upload` (the mockup fetches it; omit it
for a screenshot-pending ad and it renders a placeholder):

```bash
curl -s -X POST "https://hundrads.com/v1/reports/ad-preview" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
  -d '{"brand":"<brand>","ad_name":"<name>","primary_text":"...","headline":"...",
    "description":"...","call_to_action":"<CTA>","link_url":"https://...",
    "image_url":"<image_url from media call>","campaign_name":"<campaign>",
    "daily_budget_cents":1500,"agent_note":"<one line: what this variant tests>"}'
```

Send it ONLY for new test ads you staged this pass — not for scale duplicates of
an existing winner (the user has already seen that creative). A 422
(`no Discord webhook configured`) means the Reports tab isn't set up; treat it
exactly like the digest 422 — don't fail the pass, just note it once. These
previews are IN ADDITION to the single per-brand digest in step 5.


### 5. Report the pass via Hundrads (`POST /v1/reports/ads`)
Delivery is owned by the server. The user picks the channel + webhook/chat/email
in the dashboard **Reports** tab; the autopilot NEVER holds a webhook. POST one
structured report and Hundrads fans it out to the configured channel, rendering
the account-health KPI card + spend-vs-ROAS chart ITSELF from live Meta data — so
you supply section content only and never restate those numbers or draw charts.

**Write every word for a busy boss.** The reader is sharp but has ~20 seconds on
their phone. Use short, plain sentences a non-marketer instantly gets. No jargon,
no acronyms (say "for every RM1 spent we made RM3.60 back", not "ROAS 3.6"), no
internal terms, no hedging. Every line answers "so what — for the money or my
time?". If a smart friend who knows nothing about ads wouldn't follow it in one
read, rewrite it. Lead with the single most important thing; cut everything that
isn't decision-useful.

Send **once per brand per pass**, respecting the quiet rules:
- **Skip** when nothing changed: hash the report fields; equal to
  `state.brands[brand].last_report_hash` → send nothing.
- **Send** when a NEW fire appeared this pass (rejected / zero-delivery /
  spend-runaway / roas-crash not already in `state.alerts_sent`, key
  `"<ad_id>:<fire_type>"`), OR `now - last_report_sent >= report_interval_hours`,
  OR the daily summary hasn't gone out today.

```bash
curl -s -X POST "https://hundrads.com/v1/reports/ads" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
  -d '{
    "brand":"forge",
    "headline":"Forge is healthy. I paused 1 tired ad; sales are still coming in steadily.",
    "health":{"spend_today":12.75,"roas_today":3.6,"spend_7d":89.0,"roas_7d":3.1},
    "actions":[{"kind":"status_change","object_name":"Ad A — fatigued","summary":"Paused — it had stopped paying its way","status":"pending"}],
    "watching":["One new ad is still warming up — about a day in, too early to judge."],
    "outcomes":["Yesterday'\''s change worked: sales held steady, no drop."],
    "blocked":["When you have a moment, approve 1 ad to pause and 2 new ads."],
    "next_step":"Approve those in the dashboard — nothing else needs you."
  }'
```
Returns `{channel, ...}` on delivery. Field contract:
- `headline` (required) — one plain sentence: what happened this pass and why it
  matters. Lead with the worst problem if there is one (ad rejected / not
  spending / overspending / sales dropped); otherwise lead with the win.
- `health` — only facts the server's chart can't show; money keys auto-format RM.
  Use **real** numbers: real revenue + orders from the sales-tracker — map the
  brand via `config.sales_brand_by_brand` (`forge`→`pandaitech_forge`),
  `GET /v1/sales/summary?brand=<sales_brand>&start_date=&end_date=` (+
  `/v1/sales/daily` for the trend). **real ROAS = real_sales ÷ brand Meta spend**
  (tracker `ad_cost`/`roas` are usually 0 — ignore, divide by Meta spend
  yourself). Spend + CTR from `GET /v1/insights?brand=<brand>&level=campaign`,
  brand-scoped (account totals are NOT brand-filtered on shared accounts).
  Tracker down → fall back to Meta `Σ roas×spend`. No data yet → say so
  (`"spend_today": 0`) rather than omitting the key.
- `actions` — one entry per draft/proposal filed THIS pass: `{kind, object_name,
  summary, status}` (kind = `meta_ad` / `status_change` / `budget_change`).
- `watching` / `outcomes` — short everyday-word bullets: what's still warming up
  and not worth touching yet; whether last pass's changes actually worked.
- `blocked` — what needs the user: pending drafts, scale/kill awaiting approve,
  approved-but-unactivated PAUSED ads, screenshot-needed notes. Phrase as
  "approve … in the Hundrads dashboard". Empty list = nothing waiting.
- `next_step` — one plain sentence: the single most useful thing to do next.
Never mention the approval-flow mechanics, internal ids, or tool names in the text.
Plain words throughout: a smart boss with no marketing background should understand
every line on the first read, in seconds.

**If it 422s** (`no Discord webhook configured` / no channel set) → the user
hasn't set up the **Reports** tab yet. Don't fail the pass: keep the report in
`state` and add a one-time "set your report channel in the Hundrads dashboard →
Reports tab" note. On a successful send, record
`state.brands[brand].last_report_sent`, `last_report_hash`,
`last_daily_summary_date`, and the fires you reported into `alerts_sent`.

### 6. Persist state
Write `state.json`: `last_pass=now`, updated per-brand `staged` / `last_test_date`
/ `last_fingerprint`, `alerts_sent`, `review_queue`, `poster_spend_today` +
`poster_spend_date`, and the per-brand report fields if a report went out
(`last_report_sent`, `last_report_hash`, `last_daily_summary_date`).

## Manual / dry run
Invoked by hand: do one pass with the same logic. For a safe rehearsal, treat
`guards.max_new_tests_per_brand_per_day` as 0 (analyze + classify + write the
digest only, submit ZERO drafts or proposals) — useful to confirm wiring before
letting it submit.

## Operational notes
- Fired by a **cron** entry that runs `scripts/run_ads_skill.sh autopilot` every
  4h. The wrapper runs ONE headless pass — `claude -p "<task>"
  --dangerously-skip-permissions --model 'claude-opus-4-8[1m]'` (Opus 4.8, 1M
  context) — then exits. No tmux, no persistent window; an `flock` guards against
  two passes overlapping. Logs go wherever cron redirects (e.g. `logs/cron.log`);
  for a live view add `--output-format stream-json --verbose`.
- Change cadence by editing the cron line (`crontab -e`).
- Report delivery (channel + Discord webhook / Telegram / email) is configured by
  the user in the dashboard **Reports** tab, NOT in `.env`. The autopilot only
  calls `POST /v1/reports/ads`; the server resolves and delivers.

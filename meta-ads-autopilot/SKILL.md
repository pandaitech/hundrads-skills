---
name: meta-ads-autopilot
description: Autonomous Meta-ads marketer loop. Fired on a schedule (durable cron), it runs ONE full unattended pass — refresh insights, classify each ad, decide scale/kill/new-test, draft creative in the account voice, submit meta_ad drafts + scale/kill proposals to the Hundrads review queue, and report the pass via Hundrads (POST /v1/reports/ads → the channel set in the dashboard Reports tab). No human in the loop mid-pass; every decision resolves from config defaults. Replaces the freelance ad manager. Use when a cron prompt says "run the meta-ads-autopilot" / "one autopilot pass", or the user asks to run/test the autopilot manually.
---

# Meta Ads Autopilot

A scheduled, self-deciding wrapper around the manual [`meta-ads`](../meta-ads/SKILL.md)
capability. Each firing = **one full pass** that runs to completion with **no
human in the loop**: analyze → decide → generate creative → submit drafts →
alert. The user is not available to answer during a pass — never stop for input.

Tunables live in `.claude/autopilot/config.json` (read at the start). There is
**no local state file** — the autopilot is stateless on disk. Everything that
used to sit in `state.json` now lives in Hundrads and is viewable in the
dashboard **Agent** tab:
- **Facts** (what's running, what's pending, what you staged, today's spend) are
  DERIVED LIVE every pass — `GET /v1/agent/ops?brand=` (backlog + counters) and
  the live `/v1/ads` `/v1/drafts` reads. Never cache them; activating an ad in
  the dashboard must change your view instantly.
- **Notification dedup** ("did I already say this / has anything changed") is
  owned by the server — pass a `fingerprint` to `POST /v1/reports/ads` and use
  `POST /v1/agent/notify-check` for immediate alerts. You track nothing.
- **Durable judgment** (the only thing you persist) goes to
  `POST /v1/agent/notes` — learnings/intent, never facts. Read them back at the
  start of a pass with `GET /v1/agent/notes?brand=`.

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
- **Anti-churn.** Submit at most `guards.max_new_tests_per_brand_per_day` new
  tests per brand per day, regardless of how often the loop fires — check
  `meta_ad_drafts_today` in `GET /v1/agent/ops?brand=<brand>` (a live count, no
  stored date). Don't restage an angle you just tested; the recent drafts in
  `GET /v1/drafts?brand=<brand>` show what you touched.
- **No pile-up (NEW TESTS ONLY — never proposals).** The backlog gate exists so
  test ads don't stack faster than the user reviews them. It applies ONLY to
  staging new `meta_ad` test drafts. Proposals on live ads — `budget_change`,
  `status_change` — are one-click reversible, touch a different object, and are
  **NEVER gated by the backlog**: if a winner is due a scale or a loser is due a
  pause, file the proposal regardless of what else is pending. Before staging a
  new TEST, run BOTH checks — both derived live, so they self-clear the instant
  the user acts:
  1. `GET /v1/drafts?status=pending&brand=<brand>` — pending `meta_ad` drafts
     mean the user hasn't acted on the last test batch → no new tests.
  2. `GET /v1/ads/pending-activation?brand=<brand>` (or `pending_activation` in
     `GET /v1/agent/ops`) — ads you staged that are still PAUSED. This is the
     authoritative backlog; do NOT reconstruct it from a remembered list. The
     server already excludes PARKED ads (see next rule).
  If either is non-empty, stage no new tests this pass — note "act on the
  pending X" in the digest and carry on with the REST of the pass (classify,
  propose scales/kills, mine opportunities, report). A blocked test queue is
  never a reason to go silent or skip analysis.
- **Parked ads.** A staged ad still PAUSED more than `guards.parked_after_days`
  days after approval = the user has implicitly declined it. The server drops it
  from `pending_activation` automatically. The first pass you notice an ad newly
  parked, suggest ONCE (digest `recommendations`) that the user activate or
  delete it — then never count it against anything again. Don't re-raise parked
  ads; a durable note ("<ad> parked, already flagged") stops the repeat.
- **Cost.** A quiet INTRADAY pass makes ZERO model/poster calls. On the DAILY
  pass (the notify-check-gated full refresh), analysis is part of the job — the
  opportunity-mining step may read any data it needs; that's thinking, not
  spending. Model calls (`POST /v1/complete`) only to draft copy actually being
  submitted this pass; posters (`POST /v1/media/poster`) only for a draft being
  submitted. Stop generating posters once
  `poster_spend_today_usd` (from `GET /v1/agent/ops`) would exceed
  `guards.max_poster_cost_usd_per_day`.
  Uploading a ready-made asset (`POST /v1/media/upload`) costs nothing and does
  not count against the poster cap — prefer it once the cap is near.
- **Quiet.** Default to silence. Discord only on a true fire (immediate) or when
  the digest tick is due (see step 6). Most passes send nothing.

**Why you may decide without asking:** the draft wall. The worst outcome of any
autonomous call is a pending draft in the Hundrads review queue the user never
approves — zero Meta objects created, zero spend, fully reversible with one
Reject click. Nothing the autopilot does can reach Meta; only the user's Approve
can, and even that only ever stages PAUSED. Keep that wall intact and you cannot
cause harm.

## One pass

### 1. Load context
Read `config.json`. Compute `now` (UTC). Pull `GET /v1/agent/ops?brand=<brand>`
for the working state (pending-activation backlog, pending drafts,
`meta_ad_drafts_today`, `poster_spend_today_usd`, `last_pass`) and
`GET /v1/agent/notes?brand=<brand>` for what you learned last time. No local
state file — these reads ARE your state, always current.

### 2. Refresh data (cost-aware)
- **Daily full refresh:** once per day, on the first pass on/after
  `daily_summary_hour`. Gate it with the server: `POST /v1/agent/notify-check`
  `{scope:"daily-refresh", key:"<brand>", min_interval_min:1200}` and refresh
  only when it returns `send:true` (it records the run, so later passes today
  skip). Refresh the library for each brand in `config.brands`:
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
- `learning` — younger than `min_adset_age_hours` → leave alone.
- `winner` — out of learning, ≥ `min_results_for_judgement` results, ROAS
  clearly above brand average, healthy CTR.
- `loser` / `fatigued` — out of learning, ≥ `min_results_for_judgement`
  results, ROAS below average or CTR decayed.
- **Early-signal tier** — past `min_adset_age_hours` but still under
  `min_results_for_judgement` results. On small budgets 50 results can take
  months, so don't freeze: judge provisionally on proxies.
  - `early-loser` — spent ≥ `guards.early_kill_spend_mult` × the brand's
    typical cost-per-result (use the brand's 30d average cost per purchase
    from insights; if none exists yet, 3 full days of the ad set's daily
    budget) with **zero** results, or CTR/CPC clearly worst-in-brand →
    eligible for a pause proposal. Say "early signal" in the reason.
  - `early-standout` — clearly best-in-brand CTR/CPC **and** at least a few
    results → eligible to seed an ITERATE TEST (new variants of its angle) —
    or, when conviction is high (best converter in the brand, sustained over
    several days, not one-sale noise), a modest `budget_change` proposal
    (≤ +30%) on it instead (operator: "if you think the angle works, increase
    the budget"). A budget increase never auto-approves, so proposing early is
    safe — the user decides. Autonomous confidence stays with the 50-result
    `winner` bar; below it, propose with the "early signal" caveat in the
    reason.
  - Neither → `learning`, leave alone.

### 4. Mine opportunities (daily pass — the proactive half of the job)
Watching is not managing. Once per day (piggyback the daily-refresh
notify-check gate — when IT fires, this step runs), actively hunt for up to
`guards.max_recommendations_per_pass` patterns worth acting on. Look across
data you already have (library, insights, sales daily, comments, agent notes):
- **Angle vs angle** — one hook/angle/format beating the others on CTR or
  cost-per-result → propose testing more of it.
- **CTR–ROAS divergence** — clicks are cheap but sales don't follow → the ad
  works, the page/offer doesn't; recommend what to check.
- **Time patterns** — days/dayparts in `GET /v1/sales/daily` that convert
  reliably better → recommend concentrating tests or budget timing there.
- **Comment signals** — repeated objections or questions on ad posts are free
  audience research → propose an ad angle that answers them.
- **Untested ground** — a proven angle never tried on the other brand, a
  format gap (all statics, no video), an audience the winners share.

Every finding MUST resolve into exactly one of:
1. **A draft filed this pass** — a new test (within the cap + backlog gate) or
   a proposal. Acting beats suggesting when the data is clear.
2. **A `recommendations` bullet in the digest** — a specific plan, not an
   observation: "Comments keep asking about price — I want to test 2 ads that
   lead with the RM49 price. Approving the pending batch unblocks me." Never
   file the same recommendation two passes running (check your notes).

**Default to 1 or 2 — the user must SEE what you found.** A durable note
(`POST /v1/agent/notes`) is a supplement for carrying the judgment forward,
NEVER a substitute for showing the finding: a mining insight that ends up
only in a note is invisible and worthless to the user. On a daily pass where
mining ran, the digest must either carry ≥1 `recommendations` bullet or the
mining genuinely found nothing (rare on an active account — say so in
`watching` if so). Every draft you file from mining also gets an `actions`
entry, so the user sees WHY it exists.

### 5. Decide + act (autonomous, all defaults from `config.defaults`)
For each brand, in priority order, respecting every guardrail:

- **Winner due a scale** — out of learning, ROAS above the brand's target/avg,
  healthy CTR, and no scale staged within `restage_cooldown_hours` (the ~3-day
  cadence). Grow it with a **ROAS-sized budget bump, capped at +30%**, so Meta's
  learning phase isn't reset. Size the step by how strong the winner is:

  | Winner strength (out of learning) | Budget bump |
  |---|---|
  | Strong — ROAS ≥ 1.5× brand target/avg, healthy CTR | **+30%** |
  | Solid — ROAS ≈ 1.2–1.5× target | **+20%** |
  | Marginal — ROAS just above target (≈ 1.0–1.2×) | **+10%** |
  | At/below target, or CTR decaying | **don't scale** — hold or kill instead |

  Never exceed +30% in one step — a bigger jump resets learning and tanks the ad
  (this graduated table supersedes any flat `scale_budget_mult`). Base = the ad
  set's current daily budget (never step down); `new = round(base × (1 + pct/100))`
  to the nearest RM1.

  **Prefer editing the live ad set** with a `budget_change` proposal — a ≤30%
  edit keeps the existing learning phase, whereas a duplicate restarts it, so
  editing almost always wins:
  ```bash
  curl -s -X POST "https://hundrads.com/v1/drafts" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY" -H 'Content-Type: application/json' \
    -d '{"kind":"budget_change","agent_note":"<pitch>","payload":{
      "brand":"<brand>","adset_id":"<id>","adset_name":"<name>",
      "current_daily_budget_cents":1500,"new_daily_budget_cents":1950,
      "reason":"<why — name the ROAS tier that set the %>","expected_impact":"<what you expect>"}}'
  ```
  Only stage a higher-budget PAUSED duplicate (a `meta_ad` draft reusing the
  winner's copy + image, see "Submitting drafts" below) when a genuinely *fresh*
  ad is wanted — and note in the digest that a duplicate starts a new learning
  phase. The `budget_change` draft itself is the record that you bumped this ad
  set — don't re-bump while it's still pending in `GET /v1/drafts`. Mention the
  scale in the digest. Budget increases never auto-approve — they always wait
  for the user.
- **New test due** (under `max_new_tests_per_brand_per_day`, test backlog gate
  clear) → pick the seed in priority order: a `winner`'s angle to iterate, an
  `early-standout`'s angle, or an opportunity from step 4. **No winner or
  standout yet (cold account)** → run angle DISCOVERY instead of iteration:
  `defaults.variants_per_test` ads on genuinely different angles (see the
  `ad-testing` skill's COLD START mode), judged later on early proxy signals.
  **What counts as a test variable (operator rule):** a test must change
  something that can change a DECISION — the angle/hook, the offer framing,
  the format (static / video / carousel), or the audience. Visual execution
  alone (typography, layout, palette, a "premium re-set" of an existing ad)
  is NOT a variable: two ads saying the same thing to the same people teach
  nothing, split delivery, and slow learning. Never stage an ad whose angle
  is already live or already sitting in the queue — if you believe in that
  angle, the correct move is a `budget_change` proposal on the existing ad
  set, not a lookalike ad. Restyle ONLY when fatigue is the diagnosis (a
  proven angle whose CTR decayed) — and then the restyle REPLACES the tired
  ad (pause proposal alongside), never runs beside it.
  Draft `defaults.variants_per_test` variations in the
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
- **Loser / early-loser / fatigued / fire** → DO NOT act on Meta. File a pause
  proposal (for an `early-loser`, say "early signal" in the reason so the user
  knows it's provisional):
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
    "description":"...","call_to_action":"<CTA>","destination":"website",
    "link_url":"https://...","daily_budget_cents":1500,"image_hash":"<hash>"}}'
```

**State the ad type in every pitch — the user must never have to guess what
kind of ad they're approving.** The FIRST sentence of every `meta_ad` draft's
`agent_note`, its digest line, and its preview `agent_note` names the ad type:
"Website ad — click opens <landing page>" or "WhatsApp ad — click opens a
WhatsApp chat", plus the objective. `destination` controls it:
`"website"` (default, click → `link_url`) or `"whatsapp"` (click-to-WhatsApp:
click opens a chat with the Page's connected WhatsApp Business number;
`link_url` is ignored, the CTA auto-sets to `WHATSAPP_MESSAGE`, and
OUTCOME_SALES needs no pixel — conversations are the signal). Only draft
`whatsapp` when the brand closes sales in chat AND the Page has a WhatsApp
number connected; if the push fails at approve with a WhatsApp error, flag the
missing Page↔WhatsApp connection in the alert instead of retrying.

Placements are automatic — website ads with reach/traffic/sales objectives also
serve in the Threads feed. That's not a draft field: the Instagram and Threads
profile identities are brand-level config on the dashboard's Brands page.

Returns `{draft: {id}, warnings, review_url}`. Give every variant in the same
test the SAME `campaign_name` — on approval the handler get-or-creates the
campaign by name, so all variants land in one campaign (consolidated learning).
Resolve `objective` from the brand's top past ads (`GET /v1/library/ads` items
include `objective`); fall back to `defaults.objective_by_brand[brand]`. Use
`defaults.default_cta` and `defaults.daily_budget_cents` unless scaling. Surface
any `warnings` from the response in the digest. The draft itself never touches
Meta — only the user's Approve does, and it only ever creates PAUSED objects.
Once approved, the ad shows up in `GET /v1/ads/pending-activation` until the
user turns it on — no need to track what you staged.

**Multi-advertiser ads — opted OUT by default.** Meta's default is opt-IN, which
lets our ad appear grouped with other advertisers' in browse/discovery surfaces.
The push path (`create_paused_ad`) sets `contextual_multi_ads=OPT_OUT` on every
creative, so all pushed ads run standalone. The field is immutable post-create —
changing it on a live ad means cloning the creative (reuse the same
`object_story_id` to keep social proof) and swapping it onto the ad.

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
previews are IN ADDITION to the single per-brand digest in step 6.


### 6. Report the pass via Hundrads (`POST /v1/reports/ads`)
**The Discord report IS the product.** The user judges every pass — and the
autopilot itself — solely by this message. Work that isn't visible in it
(notes, internal reasoning, held decisions) does not exist to them, so
anything you found, filed, or decided this pass must be readable right there.
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

Send **once per brand per pass** — but let the SERVER enforce the quiet rules,
so you track nothing. Build a `fingerprint` string from the situation that
matters (e.g. `"forge-sp74.59-roas3.28-HOLD-backlog0-noactions"`) and pass it,
plus `min_interval_min` (= `report_interval_hours`×60), to `POST /v1/reports/ads`
(fields below). The server delivers only when the fingerprint **changed** or the
interval elapsed, else returns `{"sent":false,"reason":"unchanged"}`. A genuinely
new fire changes the fingerprint, so include the active fire keys in it.
For an **immediate** fire alert (separate from the digest), gate it with
`POST /v1/agent/notify-check` `{scope:"alert", key:"<ad_id>:<fire_type>"}` and
send only on `send:true` — this replaces the old local `alerts_sent` map.

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
    "recommendations":["People keep asking about the price in comments — I want to test 2 ads that lead with the RM49 price next."],
    "blocked":["When you have a moment, approve 1 ad to pause and 2 new ads."],
    "next_step":"Approve those in the dashboard — nothing else needs you.",
    "fingerprint":"forge-sp12.75-roas3.6-7d89-roas3.1-paused1-blocked3-noFires",
    "min_interval_min":240
  }'
```
Returns `{channel, ...}` on delivery, or `{"sent":false,"reason":"unchanged"}`
when the server deduped it (unchanged within the interval) — that's expected on
quiet passes, not an error. Field contract:
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
- `recommendations` — proactive suggestions from step 4 that didn't become a
  draft this pass: each a specific plan in plain words ("Comments keep asking
  about price — I want to test 2 ads leading with the RM49 price"), never a
  vague observation. Renders as its own "Suggestions" section. Empty list =
  nothing worth suggesting; don't pad it.
- `blocked` — what needs the user: pending drafts, scale/kill awaiting approve,
  approved-but-unactivated PAUSED ads, screenshot-needed notes. Phrase as
  "approve … in the Hundrads dashboard". Empty list = nothing waiting.
- `next_step` — one plain sentence: the single most useful thing to do next.
Never mention the approval-flow mechanics, internal ids, or tool names in the text.
Plain words throughout: a smart boss with no marketing background should understand
every line on the first read, in seconds.

**If it 422s** (`no Discord webhook configured` / no channel set) → the user
hasn't set up the **Reports** tab yet. Don't fail the pass; add a one-time "set
your report channel in the Hundrads dashboard → Reports tab" note. The server
records send-dedup itself (from the `fingerprint`), so there is nothing for you
to write back after a send.

### 7. Persist learnings (only)
No state file to write — facts are derived and dedup is server-side. The ONLY
thing to persist is durable JUDGMENT worth carrying to the next pass: save it
with `POST /v1/agent/notes` `{brand, text, tags?}`. Examples: "the Non-Tech
angle keeps getting disapproved — stop retrying", "Price Anchor scales cleanly
to RM47/day, CTR holds". NEVER write a fact a query can answer (ad status,
spend, what's pending) — that's what `GET /v1/agent/ops` and the live reads are
for. Delete a note (`DELETE /v1/agent/notes/{id}`) once it stops being true.

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

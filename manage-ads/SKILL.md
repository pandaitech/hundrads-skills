---
name: hundrads-manage-ads
description: Run the daily digital-marketer loop on live Meta (Facebook/Instagram) ads for any brand in the ad-account registry — check performance, detect issues, kill losers, scale winners, fix fatigued creatives, handle comments, and queue the next test. Use when the user says "check my ads", "manage my ads", "how are my ads doing", "kill/scale anything?", or a scheduled run invokes it. Everything goes through Hundrads PROPOSALS — never touches Meta directly; the user approves in the dashboard (or a standing auto-approve policy executes low-risk kinds).
---

# Manage Meta ads (the digital-marketer loop)

Do what a good ads manager does every morning: **refresh data → read the
account → learn from past verdicts → diagnose → decide → propose → report.**
You replace the human marketer's judgment, NOT their authority — every action
is a proposal in the Hundrads review queue.

> **Money safety.** This skill NEVER touches Meta for writes. All changes go
> through proposals (`POST /v1/drafts`, kinds: `budget_change`,
> `status_change`, `ad_edit`, `comment_reply`, `comment_hide`). Only the
> user's Approve in the dashboard — or a standing auto-approve rule THEY
> enabled — executes one, server-side. Budget increases, resume, and ad edits
> can never auto-approve. You cannot approve anything yourself.

## How to call the API

Every call in this skill is a curl against the Hundrads REST API:

```bash
# HUNDRADS_API_KEY = your workspace key (minted in the dashboard Keys tab)
curl -s "https://hundrads.com/v1/..." -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

POSTs add: `-X POST -H 'Content-Type: application/json' -d '{...}'`

## Setup reality (check first)

- `HUNDRADS_API_KEY` set — a per-workspace `hnd_live_…` key minted in the
  dashboard **Keys** tab.
- Reading live ads/insights needs `ads_read`; executing approved proposals
  needs `ads_management`. If a call errors on permissions, report exactly
  what's missing — don't fake numbers.
- Brands come from the registry: `GET /v1/accounts`. Never hardcode an
  account id. No brand given? Run the loop for every brand that has live ads.

## Unattended mode (cron)

This skill must work headless. When no user is present:
- Never ask questions. Apply the policy + defaults and submit proposals —
  the review queue IS the conversation.
- Skip anything that requires an in-chat decision; note it in the report.
- Always finish with the report so the morning dashboard tells the story.

## The loop

### 1. Refresh + read the account

```bash
# pull latest insights into the library
curl -s -X POST "https://hundrads.com/v1/library/refresh" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"target": "ads", "brand": "<brand>"}'

# what's running now (budget, ctr, roas, frequency)
curl -s "https://hundrads.com/v1/ads?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"

# account averages + top-quartile thresholds
curl -s "https://hundrads.com/v1/benchmarks?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"

# today's spend pace per adset
curl -s "https://hundrads.com/v1/budget/pacing?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"

# the working window
curl -s "https://hundrads.com/v1/insights?brand=<brand>&date_preset=last_7d&level=ad" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"

# daily trend when judging momentum
curl -s "https://hundrads.com/v1/insights?brand=<brand>&date_preset=last_7d&level=ad&time_increment=1" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

Optional but powerful: ground-truth revenue via

```bash
curl -s "https://hundrads.com/v1/sales/summary?brand=<brand>&start_date=<YYYY-MM-DD>&end_date=<YYYY-MM-DD>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
curl -s "https://hundrads.com/v1/sales/daily?brand=<brand>&start_date=<YYYY-MM-DD>&end_date=<YYYY-MM-DD>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

and head-to-head A/B reads via

```bash
curl -s "https://hundrads.com/v1/insights/compare?brand=<brand>&object_ids=<id1>,<id2>&date_preset=last_7d" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

### 2. Read the rules — policy first, then judgment

BEFORE deciding anything:

```bash
curl -s "https://hundrads.com/v1/policies?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

- **`management_prompt`** — the user's management rules in their own words.
  This OVERRIDES every default heuristic below. If it says "kill at RM50
  spend with no sale", that replaces your kill rule. Quote the rule you
  applied in each proposal's `agent_note`.
- `max_daily_budget_cents` / `max_budget_increase_pct` — hard caps. Never
  propose past them.
- `auto_approve` — know which of your proposals will execute without a click
  (possible: comment_reply, comment_hide, pause, budget_decrease). Mention
  it in the report.

### 3. Learn from the human before proposing

The user's past verdicts are training data — read them EVERY run:

- Recent approve/reject verdicts:

  ```bash
  curl -s "https://hundrads.com/v1/decisions?brand=<brand>" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY"
  ```

  A rejection's `review_note` is the user telling you your judgment was
  wrong. Never re-propose something equivalent to a rejection without
  addressing the note.
- Executed proposals with before/after-7d numbers:

  ```bash
  curl -s "https://hundrads.com/v1/outcomes?brand=<brand>" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY"
  ```

  Did your last scale actually hold ROAS? Did the pause save money? Say so
  in the report; adjust your thresholds when outcomes disprove them.
- **Dedupe gate**:

  ```bash
  curl -s "https://hundrads.com/v1/drafts?status=pending&brand=<brand>" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY"
  ```

  If a pending proposal already targets an object, do NOT submit another for
  the same object. One open question per object at a time.

### 4. Diagnose

Start from the alerts (delivery issues, fatigue, underperformance, pacing):

```bash
curl -s "https://hundrads.com/v1/alerts?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

then verify each alert against the data yourself — alerts are leads,
not verdicts. Add your own scan:

- **Fatigue**: frequency > 3 AND CTR trending down over the daily series.
- **CPM spike**: CPM well above account average without a CTR gain (auction
  pressure or creative rejection — check `effective_status`).
- **Delivery stalls**: budget present, near-zero spend today (pacing row).
- **Learning phase**: adset younger than ~3 days or under ~50 conversions in
  its window. **Protected — don't touch it** unless it's bleeding with zero
  results well past the kill threshold.
- **Unanswered comments**:

  ```bash
  curl -s "https://hundrads.com/v1/comments?brand=<brand>&unanswered=true" \
    -H "Authorization: Bearer $HUNDRADS_API_KEY"
  ```

  Comments on ads are purchase-intent signals and social proof rot when
  ignored.

### 5. Decide — default heuristics (management_prompt overrides ALL of this)

Calibrate to the account itself via the benchmarks endpoint, not industry
folklore. Let spend×AOV define "enough data": estimate target CPA from the
brand's brief/sales data when available.

| Situation | Default action |
|---|---|
| Spend ≥ ~3× target CPA, zero conversions, out of learning | **Kill** — `status_change` pause on the ad (adset if all its ads qualify) |
| CTR < ~half account average after meaningful spend (≥ RM30) | **Kill or fix** — pause, or `ad_edit` if the offer is right but the hook is weak |
| ROAS ≥ top-quartile for 3+ consecutive days, stable CPA | **Scale up** — `budget_change` +20% (never more than `max_budget_increase_pct`, never past `max_daily_budget_cents`; one step per day per adset) |
| ROAS positive but declining + frequency > 3 | **Refresh** — propose creative refresh: `ad_edit` for copy, or hand off to the meta-ads skill for a new variant |
| ROAS between break-even and target | **Scale down** — `budget_change` −20–30%; cheaper than killing a learner |
| Genuine buying-intent comment | `comment_reply` in brand voice (read the brand brief first — `GET /v1/brand/brief?brand=<brand>`) |
| Spam/abuse comment | `comment_hide` |

Rules of engagement:
- **One change per object per day.** Meta re-enters learning on big edits;
  batch your reasoning, not your changes.
- Scale winners by raising budget, not by duplicating (duplication resets
  learning) — unless the user's prompt says otherwise.
- Prefer pausing an AD over an ADSET over a CAMPAIGN — smallest blast radius
  that solves the problem.
- When the data is ambiguous, propose nothing and say why in the report.
  A marketer who churns changes daily is worse than one who waits a day.

### 6. Submit proposals

For each decision, POST a proposal:

```bash
curl -s -X POST "https://hundrads.com/v1/drafts" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"kind": "<kind>", "agent_note": "<note>", "payload": {...}}'
```

`payload` per kind:

- `budget_change`: `{brand, adset_id, adset_name, current_daily_budget_cents,
  new_daily_budget_cents, reason, expected_impact}`
- `status_change`: `{brand, object_type: ad|adset|campaign, object_id,
  object_name, action: pause|resume, reason}`
- `ad_edit`: `{brand, ad_id, ad_name, primary_text, headline, link_url,
  call_to_action, reason}`
- `comment_reply`: `{brand, comment_id, message, comment_text, commenter}` /
  `comment_hide`: `{brand, comment_id, comment_text, reason}`

**`reason` is your pitch — you are the marketer, the user is the boss.**
The dashboard shows the live object (creative, budget, track record) next to
your reason; your job is the part the data can't say: what you noticed, what
you think it means, and what you want to do about it. One short paragraph,
first person, full sentences. The three shapes that work:

- *Winner*: "This angle did RM4,100 revenue on RM812 spend (ROAS 5.05) over
  the last 7 days — I want to push it further with a fresh creative before
  it fatigues."
- *Loser with a hypothesis*: "This angle didn't convert (RM68 spent, no
  sales), but I think the creative is the problem — it looks too corporate
  for our audience. I propose retesting the same angle with a more casual
  creative at a smaller budget."
- *No data yet*: "I don't have performance data for this audience yet, so I
  suggest we try these 3 angles at RM20/day each and let the numbers decide."

Hard rules for `reason` (and `agent_note`):

- **Never mention the approval flow.** No "needs your approve to spend", no
  "executes once approved" — the user knows how their own queue works.
- **No internal codenames or shorthand.** Not "Cell 3", not "angle × hook",
  not "ACTIVATE INVENTORY (8/8)". Describe the thing in the user's words:
  "the price-anchor ad in the Skip Video test".
- Numbers are evidence inside the story, not a data dump. Quote a
  `management_prompt` rule only when it's the actual trigger.
- If a sentence doesn't help the boss decide, delete it.
- **Chained proposals** (campaign + adset + ad to launch one test): the full
  pitch goes on the MAIN object; each parent gets one plain line — "Just
  unlocks the price-anchor test below; spends nothing by itself."

`expected_impact` (budget_change): one-sentence forecast — "expect ~RM150/day
revenue if ROAS holds". `agent_note`: only context that doesn't fit `reason`;
empty is fine — never a restatement. The response is
`{"draft": {"id", "status"}, "warnings": [...], "review_url": "..."}` —
relay the returned `warnings` and the `review_url` to the user.

### 7. Next test — fires on ANY of these, not just "test over"

Check every run, in this order:

0. **Read the brand brief first** (skip only if the user's request is
   self-contained):

   ```bash
   curl -s "https://hundrads.com/v1/brand/brief?brand=<brand>" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Brands come from `GET /v1/accounts`. The brief carries the user's voice
   notes and channel playbooks — follow them over any generic style. Also
   study past posts in the library endpoints to match what actually performed.

1. **Paused-test backlog first.**

   ```bash
   curl -s "https://hundrads.com/v1/ads?brand=<brand>&status=paused" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Ads the user approved through the queue but never enabled are READY-MADE
   inventory. Before drafting anything new: rank the backlog against the
   current winner's angle and either (a) file `status_change` resume
   proposals for the cells worth running (resume never auto-approves —
   the user's Approve is the gate), or (b) recommend culling stale cells.
   Never draft new variants while a relevant backlog sits unlaunched.
2. **A kill was proposed this run** → the account is losing a slot. Say what
   should fill it: a backlog cell, or a new variant brief (one line: angle ×
   scene × what it tests).
3. **Clear winner + clear loser exist** → the test IS concluded even if
   nobody declared it. Propose the next iteration from the winner's angle —
   backlog cell if one matches, else hand off to the **meta-ads** skill to draft.

The meta-ads skill owns drafting/creative; this skill owns the live account and
the resume/kill decisions on existing objects. Always state the budget
impact of what you're suggesting (RM/day added if enabled).

### 8. Report

End every run with a marketer's stand-up, per brand:

- **Account health**: spend, blended ROAS, vs benchmarks, pacing.
- **Actions proposed**: each with one-line reasoning + which will
  auto-approve under standing policy.
- **Watching**: things you deliberately did NOT touch (learning phase,
  ambiguous trends) and what would change your mind.
- **Outcome check**: how earlier executed proposals performed
  (`GET /v1/outcomes`).
- **Blocked/needs human**: missing permissions, empty brief, anything
  skipped in unattended mode.

## Deliver the report

Send the SAME stand-up to the user's channel — `POST /v1/reports/ads`
exactly once per brand. The server owns the layout (Discord embed / Telegram
album / email, per-user setting) and renders the account-health dashboard as
IMAGES from live Meta data (KPI tiles for last 7d / yesterday / today + the
daily spend-vs-ROAS chart); you only fill the sections:

```bash
curl -s -X POST "https://hundrads.com/v1/reports/ads" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "brand": "<brand>",
    "headline": "2 kills filed, winner scaled +20%",
    "actions": [{"kind": "budget_change", "object_name": "Price Anchor",
                 "summary": "+20% to RM31.20/day — ROAS 4.81 over 7d",
                 "status": "pending"}],
    "watching": ["Coach Skip Video at ROAS 2.85 — one more strong day and I scale it"],
    "outcomes": ["Yesterday'\''s budget cut held: ROAS recovered 1.06 → 1.31"],
    "blocked": ["Comment inbox unreadable — Meta needs a Page access token"],
    "next_step": "Approve the queue and the account adds ~RM6/day on the winner",
    "creative_url": "<image url of the run'\''s headline creative, optional>",
    "health": {"pacing": "all adsets on pace"}
  }'
```

`headline` is ONE sentence, the outcome. `health` is OPTIONAL — see below.

Discipline — this is what keeps reports consistent run to run:

- Every claim sits in its section; nothing outside the schema. If a fact
  doesn't fit a section, it doesn't belong in the report.
- Spend/ROAS/sales numbers are drawn server-side into the dashboard images —
  do NOT restate them in `health`. Use `health` only for facts the images
  can't show (pacing note, benchmark context). Empty/omitted is the norm.
- One entry per proposal in `actions`; skip sections that are empty.
- Plain language, no internal ids, no tool names, no approval-flow talk.
- Skipped/failed report delivery never blocks the run — note it in chat
  and move on.

---
name: manage-meta-ads
description: Run the daily digital-marketer loop on live Meta (Facebook/Instagram) ads for any brand in the ad-account registry — check performance, detect issues, kill losers, scale winners, fix fatigued creatives, handle comments, and queue the next test. Use when the user says "check my ads", "manage my ads", "how are my ads doing", "kill/scale anything?", or a scheduled run invokes it. Everything goes through Hundrads PROPOSALS — never touches Meta directly; the user approves in the dashboard (or a standing auto-approve policy executes low-risk kinds).
---

# Manage Meta ads (the digital-marketer loop)

Do what a good ads manager does every morning: **refresh data → read the
account → learn from past verdicts → diagnose → decide → propose → report.**
You replace the human marketer's judgment, NOT their authority — every action
is a proposal in the Hundrads review queue.

> **Money safety.** This skill NEVER touches Meta for writes. All changes go
> through `create_proposal` (kinds: `budget_change`, `status_change`,
> `ad_edit`, `comment_reply`, `comment_hide`). Only the user's Approve in the
> dashboard — or a standing auto-approve rule THEY enabled — executes one,
> via `server/workers/executors.py`. Budget increases, resume, and ad edits
> can never auto-approve. You cannot approve anything yourself.

## Setup reality (check first)

- Hundrads server running: `.venv/bin/uvicorn server.main:app --port 7007`,
  `HUNDRADS_API_KEY` set in `.env`.
- Reading live ads/insights needs `ads_read`; executing approved proposals
  needs `ads_management`. If a tool errors on permissions, report exactly
  what's missing — don't fake numbers.
- Brands come from the registry: `list_ad_accounts` (MCP). Never hardcode an
  account id. No brand given? Run the loop for every brand that has live ads.

## Unattended mode (cron)

This skill must work headless. When no user is present:
- Never ask questions. Apply the policy + defaults and submit proposals —
  the review queue IS the conversation.
- Skip anything that requires an in-chat decision; note it in the report.
- Always finish with the report + `log_action` so the morning dashboard
  tells the story.

## The loop

### 1. Refresh + read the account

```
fetch_ads_update(brand)          # pull latest insights into the library
live_ads(brand)                  # what's running now (budget, ctr, roas, frequency)
ad_benchmarks(brand)             # account averages + top-quartile thresholds
budget_pacing(brand)             # today's spend pace per adset
ad_insights(brand, date_preset="last_7d", level="ad")   # the working window
ad_insights(brand, date_preset="last_7d", level="ad", time_increment="1")  # daily trend when judging momentum
```

Optional but powerful: `sales_summary` / `sales_daily` for ground-truth
revenue, and `compare_ads(brand, object_ids, date_preset)` for head-to-head
A/B reads.

### 2. Read the rules — policy first, then judgment

`get_policies(brand)` BEFORE deciding anything:

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

- `list_decisions(brand)` — recent approve/reject verdicts. A rejection's
  `review_note` is the user telling you your judgment was wrong. Never
  re-propose something equivalent to a rejection without addressing the note.
- `list_outcomes(brand)` — executed proposals with before/after-7d numbers.
  Did your last scale actually hold ROAS? Did the pause save money? Say so
  in the report; adjust your thresholds when outcomes disprove them.
- `list_ad_drafts(status="pending", brand=brand)` — **dedupe gate**: if a
  pending proposal already targets an object, do NOT submit another for the
  same object. One open question per object at a time.

### 4. Diagnose

Start from `ad_alerts(brand)` (delivery issues, fatigue, underperformance,
pacing), then verify each alert against the data yourself — alerts are leads,
not verdicts. Add your own scan:

- **Fatigue**: frequency > 3 AND CTR trending down over the daily series.
- **CPM spike**: CPM well above account average without a CTR gain (auction
  pressure or creative rejection — check `effective_status`).
- **Delivery stalls**: budget present, near-zero spend today (pacing row).
- **Learning phase**: adset younger than ~3 days or under ~50 conversions in
  its window. **Protected — don't touch it** unless it's bleeding with zero
  results well past the kill threshold.
- **Unanswered comments**: `ad_comments(brand, unanswered=True)` — comments
  on ads are purchase-intent signals and social proof rot when ignored.

### 5. Decide — default heuristics (management_prompt overrides ALL of this)

Calibrate to the account itself via `ad_benchmarks`, not industry folklore.
Let spend×AOV define "enough data": estimate target CPA from the brand's
brief/sales data when available.

| Situation | Default action |
|---|---|
| Spend ≥ ~3× target CPA, zero conversions, out of learning | **Kill** — `status_change` pause on the ad (adset if all its ads qualify) |
| CTR < ~half account average after meaningful spend (≥ RM30) | **Kill or fix** — pause, or `ad_edit` if the offer is right but the hook is weak |
| ROAS ≥ top-quartile for 3+ consecutive days, stable CPA | **Scale up** — `budget_change` +20% (never more than `max_budget_increase_pct`, never past `max_daily_budget_cents`; one step per day per adset) |
| ROAS positive but declining + frequency > 3 | **Refresh** — propose creative refresh: `ad_edit` for copy, or hand off to run-meta-ads for a new variant |
| ROAS between break-even and target | **Scale down** — `budget_change` −20–30%; cheaper than killing a learner |
| Genuine buying-intent comment | `comment_reply` in brand voice (read `get_brand_brief` first) |
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

For each decision, `create_proposal(kind, payload, agent_note)`:

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
empty is fine — never a restatement. Relay returned `warnings` + `review_url`
(http://localhost:7007).

### 7. Next test — fires on ANY of these, not just "test over"

Check every run, in this order:

0. **Read the brand brief first** (skip only if the user's request is
   self-contained):

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/brand/brief?brand=<brand>" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Brands come from `GET /v1/accounts`. The brief carries the user's voice
   notes and channel playbooks — follow them over any generic style. Also
   study past posts in the library endpoints to match what actually performed.

1. **Paused-test backlog first.** `live_ads(brand, status="paused")` — ads
   the user approved through the queue but never enabled are READY-MADE
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
   backlog cell if one matches, else hand off to **run-meta-ads** to draft.

run-meta-ads owns drafting/creative; this skill owns the live account and
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
  (`list_outcomes`).
- **Blocked/needs human**: missing permissions, empty brief, anything
  skipped in unattended mode.

## Log the work (client report)

```
log_action(skill="manage-meta-ads", channel="ads", brand="<brand>",
           title="Daily ads check: 2 kills, 1 scale proposed",
           summary="<one plain sentence on what + why>",
           outcome="proposed",            # or "report_only" when no action
           status="awaiting_review",      # or "info"
           deliverable="")
```

Tag any `complete` calls with
`complete(..., skill="manage-meta-ads", channel="ads", brand="<brand>")`.

## Deliver the report (send_ads_report)

After `log_action`, send the SAME stand-up to the user's channel — call
`send_ads_report` exactly once per brand. The server owns the layout
(Discord embed / Telegram album / email, per-user setting) and renders the
account-health dashboard as IMAGES from live Meta data (KPI tiles for last
7d / yesterday / today + the daily spend-vs-ROAS chart); you only fill the
sections:

```
send_ads_report(
    brand="<brand>",
    headline="2 kills filed, winner scaled +20%",   # ONE sentence, the outcome
    actions=[{"kind": "budget_change", "object_name": "Price Anchor",
              "summary": "+20% to RM31.20/day — ROAS 4.81 over 7d",
              "status": "pending"}],
    watching=["Coach Skip Video at ROAS 2.85 — one more strong day and I scale it"],
    outcomes=["Yesterday's budget cut held: ROAS recovered 1.06 → 1.31"],
    blocked=["Comment inbox unreadable — Meta needs a Page access token"],
    next_step="Approve the queue and the account adds ~RM6/day on the winner",
    creative_url="<image url of the run's headline creative, optional>",
    health={"pacing": "all adsets on pace"},   # OPTIONAL — see below
)
```

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

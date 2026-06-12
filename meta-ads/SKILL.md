---
name: hundrads-meta-ads
description: Plan, draft, and iterate Meta (Facebook/Instagram) ads through Hundrads — the safe middle layer. The agent studies the account's own past ad performance, drafts copy + creative, and submits DRAFTS to Hundrads for the user to approve in the dashboard. The agent never touches Meta directly and nothing can spend money. Use when the user wants ad copy, a campaign, an A/B test, or a performance review.
---

# Run Meta ads through Hundrads

You are working through **Hundrads** (hundrads.com): every write you make is a
draft in the user's Hundrads review queue, never a live change. The loop:

**study winners → draft variations → submit drafts → user approves/rejects →
read feedback + insights → iterate.**

> **Safety model.** You cannot touch the user's Meta account. Drafts go to
> Hundrads; only the user's explicit Approve pushes to Meta — and even then
> every object is created PAUSED. You also cannot un-reject, edit, or approve
> drafts yourself. Don't try; don't ask the user for their Meta token.

## Setup

The user has a Hundrads API key. Expect it in the `HUNDRADS_API_KEY`
environment variable, and the server base URL in `HUNDRADS_BASE_URL`
(default `http://localhost:7007`). Every request:

```bash
curl -s "$HUNDRADS_BASE_URL/v1/accounts" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

Full endpoint reference: `$HUNDRADS_BASE_URL/llms.txt` (markdown API reference). The server
also serves interactive docs at `$HUNDRADS_BASE_URL/docs`.

## Steps

0. **Read the brand brief first** (skip only if the user's request is
   self-contained):

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/brand/brief?brand=<brand>" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Brands come from `GET /v1/accounts`. The brief carries the user's voice
   notes and channel playbooks — follow them over any generic style. Also
   study past posts in the library endpoints to match what actually performed.

1. **Pick brand + objective.** `GET /v1/accounts` lists the user's brands.
   A brand with `can_push: false` can be researched and drafted for, but
   approval can't push yet — say so up front. Map the goal to an objective
   (OUTCOME_TRAFFIC, OUTCOME_SALES, OUTCOME_ENGAGEMENT, OUTCOME_LEADS).

2. **Study what worked — always, before writing a word.**

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/library/ads?brand=<brand>&sort=spend&limit=10" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Rank by `spend` (what the user trusted with money), then cross-check
   `ctr` and `roas`. Note the winning angle, benefit structure, CTA. Reuse
   the *structure* of winners; don't reinvent what already converts.
   For the live picture: `GET /v1/insights?brand=<brand>&date_preset=last_30d`.

3. **Draft 2–4 variations** for an A/B test. Each needs `primary_text`
   (benefit stack, in the voice the library shows), `headline` (the offer,
   ~40 chars), `call_to_action`, `link_url`. Change ONE variable per variant
   (hook angle OR offer framing OR CTA) so the test reads cleanly.

4. **Show the user the copy in chat first.** Iterate until they're happy.
   Don't submit drafts of copy the user hasn't seen.

5. **Creative (optional).** `POST /v1/media/poster` generates an ad image;
   pass `brand` so it uploads to the ad account and returns `image_hash` +
   `image_url` for the draft. Scene-led images: `complexity: "simple"`.
   Accurate in-image text or photorealism: `complexity: "complex"` (pricier).
   If the punchline lives in rendered screenshot text, don't AI-generate it —
   give the user the exact text to screenshot on a real phone.

6. **Submit drafts.**

   ```bash
   curl -s -X POST "$HUNDRADS_BASE_URL/v1/drafts" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
     -d '{
       "kind": "meta_ad",
       "agent_note": "Variant A of 3 — tests outcome-led hook. Modeled on the top-spend ad (RM2.1k, 3.2% CTR) which leads with the offer.",
       "payload": {
         "brand": "<brand>",
         "campaign_name": "<one shared campaign name for the whole test>",
         "ad_name": "<variant name>",
         "objective": "OUTCOME_TRAFFIC",
         "primary_text": "...",
         "headline": "...",
         "call_to_action": "LEARN_MORE",
         "link_url": "https://...",
         "daily_budget_cents": 1000,
         "image_hash": "<from poster, or empty>",
         "image_url": "<from poster, or empty>"
       }
     }'
   ```

   - Use the SAME `campaign_name` across variants of one test — Hundrads
     consolidates them under one campaign on approval.
   - `daily_budget_cents` is the account's minor unit (RM10/day = 1000).
     Confirm the budget with the user before submitting.
   - **`agent_note` is your pitch — you are the marketer, the user is the
     boss.** First person, full sentences, one short paragraph: what you
     studied (numbers inline as evidence), what this variant tests, what you
     expect. "The outcome-led hook is our top spender (RM2.1k, 3.2% CTR), so
     this variant leads with the offer — I expect it to beat the curiosity
     hook." No internal shorthand, and never mention the approval flow — the
     user knows how their queue works.
   - Relay the response's `warnings` and `review_url` to the user, then stop.
     Your job ends at a well-argued draft.

7. **Read the verdicts before the next round.**

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/drafts?status=rejected&limit=10" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   A rejection's `review_note` is the user telling you what to change —
   apply it, never resubmit an unchanged draft. `status=published` rows
   carry the PAUSED Meta ids the user can enable in Ads Manager.

8. **Iterate on performance.** After ads have run:
   `GET /v1/insights?brand=<brand>&level=ad&date_preset=last_7d` — identify
   the winner, then draft the scale-up (higher budget, new draft) and
   recommend killing losers. Recommendations in chat; changes as drafts.

## Etiquette (what makes an agent good at this)

- Research before writing: never draft from a blank page when the library
  has spend data.
- One variable per variant; say what each tests.
- `agent_note` always filled, specific, honest.
- Rejected ≠ retry. Rejected = read note, change approach.
- Never ask for or handle Meta credentials. Hundrads is the only door.

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
environment variable. The API base URL is `https://hundrads.com`. Every request:

```bash
curl -s "https://hundrads.com/v1/accounts" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

Full endpoint reference: `https://hundrads.com/llms.txt` (markdown API reference). The server
also serves interactive docs at `https://hundrads.com/docs`.

## Steps

0. **Read the brand brief first** (skip only if the user's request is
   self-contained):

   ```bash
   curl -s "https://hundrads.com/v1/brand/brief?brand=<brand>" \
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
   curl -s "https://hundrads.com/v1/library/ads?brand=<brand>&sort=spend&limit=10" \
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

5. **Creative (optional) — one image, or multiple ratios?** Before generating,
   ask the user which they want (skip the question only if they've already
   said):

   - **One image** — a single square (`1:1`) or feed (`4:5`) creative. Simplest;
     Meta shows the same image across every placement.
   - **Multiple ratios** — the SAME ad with a feed crop (`1:1` or `4:5`) *and* a
     vertical `9:16` for Stories/Reels. Approve pushes ONE placement-customized
     creative and Meta serves the right crop per placement. Best when the ad
     runs across feed + Stories/Reels — each ratio is composed separately, so
     the Stories creative uses the full vertical canvas instead of a
     letterboxed square. The dashboard preview shows every ratio so the user
     reviews each crop before approving.

   Two ways to get an `image_hash` + `image_url` for the draft — both upload to
   the brand's ad account, so pass `brand`:

   - **Generate with AI** — `POST /v1/media/poster`. Scene-led images:
     `complexity: "simple"`. Accurate in-image text or photorealism:
     `complexity: "complex"` (pricier). If the punchline lives in rendered
     screenshot text, don't AI-generate it — give the user the exact text to
     screenshot on a real phone.
     **For multiple ratios:** call poster ONCE PER RATIO with the same creative
     idea but the `aspect` tuned per slot (`"4:5"` or `"1:1"` for feed, `"9:16"`
     for Stories/Reels), and adjust the prompt's composition per ratio (e.g.
     keep the subject centered with headroom on `9:16`). Pair each returned
     `{image_hash, image_url}` with the `ratio` you asked for, for the `images`
     list in step 6.
     **BYOK:** generation uses the workspace's own AI key (`simple` → Gemini,
     `complex` → OpenAI). A **400 `No <provider> API key configured`** means none
     is stored — have the user add one at `https://hundrads.com/providers`, then retry.
     **Slow call:** a poster call can take **up to 5 min** — set the Bash `timeout`
     to **600000** (10 min) so it isn't killed by the 2-min default; not hung, just
     generating.

   - **Upload the user's own image** — when they hand you a file (product photo,
     a designed creative) instead of wanting one generated: `POST /v1/media/upload`,
     multipart `brand` + `file` (PNG/JPEG/GIF/WEBP, ≤30MB). Returns the same
     `image_hash` + `image_url`. No AI key needed. For multiple ratios, upload
     each ratio file separately and pair each returned hash with its `ratio`.

     ```bash
     curl -s -X POST "https://hundrads.com/v1/media/upload" \
       -H "Authorization: Bearer $HUNDRADS_API_KEY" \
       -F brand=<brand> -F file=@/path/to/creative.jpg
     ```

     The user can also upload or replace the image themselves on the draft's
     review card in the dashboard before approving — so submitting a draft with
     an empty `image_hash` is fine when they'd rather attach their own image.

6. **Submit drafts.**

   ```bash
   curl -s -X POST "https://hundrads.com/v1/drafts" \
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
         "image_hash": "<from poster or upload, or empty>",
         "image_url": "<from poster or upload, or empty>",
         "geo_countries": ["MY"]
       }
     }'
   ```

   - **Multiple ratios:** instead of `image_hash`/`image_url`, send `images` — a
     list of `{ "image_hash", "image_url", "ratio" }`, one per ratio (ratios
     must be distinct; valid: `"1:1"`, `"4:5"`, `"9:16"`, `"16:9"`). Two or more
     → Hundrads pushes a single **placement-customized** creative (feed gets the
     square/portrait, Stories/Reels get the `9:16`). One variant is treated as a
     plain single-image ad — just use `image_hash` instead. Example:

     ```json
     "images": [
       { "image_hash": "<feed hash>",  "image_url": "<feed url>",  "ratio": "4:5" },
       { "image_hash": "<story hash>", "image_url": "<story url>", "ratio": "9:16" }
     ]
     ```

   - **Targeting is mostly the creative's job now.** Meta's delivery finds the
     buyer from the ad itself — don't hand-build narrow interest/demographic
     audiences; let the hook and image do the targeting. The ONE lever worth
     setting is **location**, and only when the offer is geo-bound:
     `geo_countries` is a list of ISO 3166-1 alpha-2 codes. Set it when the
     product ships to one country (`["MY"]`), or a few (`["MY","SG"]`). Leave it
     off otherwise — empty defaults to the account's country. (Sub-country
     targeting — a cafe in one district — isn't supported yet; note it to the
     user if they ask.)

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
   curl -s "https://hundrads.com/v1/drafts?status=rejected&limit=10" \
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

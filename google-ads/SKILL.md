---
name: hundrads-google-ads
description: Plan, draft, and iterate Google Search ads through Hundrads — the safe middle layer. The agent studies the account's own Search performance, plans keywords, drafts responsive search ads (RSAs), and submits DRAFTS to Hundrads for the user to approve in the dashboard. It also mines the search-terms report into negative-keyword proposals. The agent never touches Google Ads directly and nothing can spend money. Use when the user wants Google ads, search ads, a keyword plan, negative keywords, or a Search performance review.
---

# Run Google Search ads through Hundrads

You are working through **Hundrads** (hundrads.com): every write you make is a
draft in the user's Hundrads review queue, never a live change. The loop:

**study winners → plan keywords → draft RSAs → submit drafts → user
approves/rejects → read feedback + search terms → iterate.**

> **Safety model.** You cannot touch the user's Google Ads account. Drafts go
> to Hundrads; only the user's explicit Approve pushes to Google — and even
> then the whole chain (campaign → ad group → ad) is created PAUSED. You also
> cannot un-reject, edit, or approve drafts yourself. Don't try; don't ask the
> user for their Google credentials.

**Know which game you're playing.** Meta is demand *creation* — the ad
interrupts someone who wasn't looking. Google Search is demand *capture* — the
person already typed what they want. So don't write curiosity hooks: match the
words in their head. The keyword tells you the intent; the ad's job is to
confirm "yes, this is that" faster than the other nine results.

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
   notes and channel playbooks — follow them over any generic style.

1. **Pick the brand.** `GET /v1/accounts` lists the user's brands. Google
   readiness is `google_can_push`: `false` means you can research and draft,
   but a `google_search_ad` draft needs the brand wired to a Google customer
   id first — say so up front instead of submitting a draft that will 422.

2. **Study what worked — always, before writing a word.**

   ```bash
   curl -s "https://hundrads.com/v1/library/ads?brand=<brand>&platform=google&sort=spend&limit=10" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Past RSAs with real performance (headlines, descriptions, spend, ctr,
   roas). If it's empty, sync first: `POST /v1/library/refresh` with
   `{"target": "ads", "platform": "google", "brand": "<brand>"}`. For the
   live picture: `GET /v1/ads?brand=<brand>&platform=google` (live rows with
   a `bucket` — running/paused — and the ids proposals need) and
   `GET /v1/insights?brand=<brand>&platform=google&level=campaign`. What
   "good" means HERE: `GET /v1/benchmarks?brand=<brand>&platform=google`.
   A fresh account has no history — then the brief and the landing page are
   your sources, and you say the numbers are projections.

3. **Plan keywords before copy.** 5–15 keywords per ad group, one intent per
   ad group (don't mix "buy X" with "what is X").

   - **Start PHRASE and EXACT.** BROAD match without conversion-based
     bidding burns money on loose queries — only reach for BROAD once the
     account records conversions and bids `MAXIMIZE_CONVERSIONS` (the draft
     API warns on BROAD + MAXIMIZE_CLICKS for exactly this reason).
   - Keep each keyword ≤80 chars and under 10 words — Google rejects longer.
   - Seed obvious negatives with the draft (`negative_keywords`): "free",
     "jobs", "course" — whatever the offer is NOT.
   - On an account with history, read what people actually typed:
     `GET /v1/search-terms?brand=<brand>` — converting terms are keyword
     candidates, junk terms are negatives (step 8).

4. **Draft the RSA copy.** Google assembles combinations from your assets and
   rates "ad strength" on variety — give it real variety:

   - **8–15 distinct headlines**, each ≤30 chars. Mix the angles: benefit
     ("Launch In 5 Minutes"), offer ("Free 14-Day Trial"), brand, proof,
     CTA ("Start Today"), and 2–3 that echo the keywords verbatim —
     searchers click the ad that repeats their words. Duplicates are dropped
     case-insensitively at submit (a warning tells you; <3 survivors is an
     error).
   - **3–4 descriptions**, each ≤90 chars — full-sentence benefit + CTA.
   - `path1`/`path2` (≤15 chars each) are the cosmetic display URL
     (`domain.com/path1/path2`) — use the keyword theme.
   - Text only. There is no image slot on a Search ad — never route the
     user toward creative generation or screenshots for this channel.
   - 2–4 variants make a test; change ONE variable per variant (angle OR
     offer framing OR keyword echo) so the test reads cleanly.

5. **Show the user the copy in chat first.** Headlines, descriptions, and the
   keyword plan with match types. Iterate until they're happy — don't submit
   drafts of copy the user hasn't seen.

6. **Submit drafts.**

   ```bash
   curl -s -X POST "https://hundrads.com/v1/drafts" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
     -d '{
       "kind": "google_search_ad",
       "agent_note": "Variant A of 2 — headlines echo the top converting search terms. The library shows offer-led copy beating brand-led (3.1% vs 1.8% CTR), so this leads with the trial.",
       "payload": {
         "brand": "<brand>",
         "campaign_name": "<one shared campaign name for the whole test>",
         "ad_group_name": "<the intent this group targets>",
         "headlines": ["...", "...", "..."],
         "descriptions": ["...", "..."],
         "path1": "offer", "path2": "trial",
         "final_url": "https://...",
         "keywords": [
           {"text": "digital product builder", "match_type": "PHRASE"},
           {"text": "create online course", "match_type": "EXACT"}
         ],
         "negative_keywords": [{"text": "free", "match_type": "PHRASE"}],
         "daily_budget_cents": 1000,
         "geo_countries": ["MY"],
         "bidding_strategy": "MAXIMIZE_CLICKS"
       }
     }'
   ```

   - Use the SAME `campaign_name` across variants of one test — campaign and
     ad group are get-or-create by exact name, so variants consolidate.
   - `daily_budget_cents` is the account's minor unit (RM10/day = 1000).
     Confirm the budget with the user before submitting.
   - `geo_countries` (ISO alpha-2) only when the offer is geo-bound; it
     applies when the campaign is newly created.
   - `bidding_strategy`: `MAXIMIZE_CLICKS` needs no conversion data — the
     launch default. Only draft `MAXIMIZE_CONVERSIONS` when the user confirms
     conversions are tracked in the account.
   - **`agent_note` is your pitch — you are the marketer, the user is the
     boss.** First person, full sentences, one short paragraph: what you
     studied (numbers inline as evidence), what this variant tests, what you
     expect. No internal shorthand, and never mention the approval flow — the
     user knows how their queue works.
   - Relay the response's `warnings` and `review_url` to the user, then stop.
     Your job ends at a well-argued draft.

7. **Read the verdicts before the next round.**

   ```bash
   curl -s "https://hundrads.com/v1/drafts?status=rejected&limit=10" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   A rejection's `review_note` is the user telling you what to change —
   apply it, never resubmit an unchanged draft. `status=published` rows carry
   the PAUSED Google ids the user can enable in Google Ads (or via the
   dashboard's Push start).

8. **Mine search terms into negatives.** Once ads have served, this is the
   highest-leverage maintenance on the channel:

   ```bash
   curl -s "https://hundrads.com/v1/search-terms?brand=<brand>&date_preset=last_30d" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Terms with clicks and zero `results` are money leaks. Propose them —
   with the numbers — as a `google_negative_keyword` draft:

   ```json
   {"kind": "google_negative_keyword", "payload": {
     "brand": "<brand>", "campaign_id": "<from /v1/ads?platform=google>",
     "keywords": [{"text": "free", "match_type": "PHRASE"}],
     "reason": "search terms: 'free ...' queries took 312 clicks / RM64 with 0 conversions in 30d"}}
   ```

9. **Iterate on performance.** After ads have run:
   `GET /v1/insights?brand=<brand>&platform=google&level=ad&date_preset=last_7d`
   — identify the winner, then propose the follow-through as drafts: a
   `google_budget_change` to scale it (read `GET /v1/policies?brand=` first —
   stay under the caps), a `google_status_change` pause for the losers, new
   RSA variants on the winning angle. Recommendations in chat; changes as
   drafts. Resume and budget increases always need the user's click — argue
   them well or don't file them.

## Etiquette (what makes an agent good at this)

- Research before writing: never draft from a blank page when the library
  and search-terms report have real data.
- Match the query. Search copy that ignores the keyword loses to copy that
  repeats it.
- One variable per variant; say what each tests.
- One intent per ad group — keyword soup makes every headline generic.
- `agent_note` and every proposal `reason` always filled, specific, honest,
  with the numbers.
- Rejected ≠ retry. Rejected = read note, change approach.
- Never ask for or handle Google credentials. Hundrads is the only door.

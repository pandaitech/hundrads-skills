---
name: hundrads-seed-page
description: Make a page/account look alive before ads send traffic to it — a short burst of organic-looking post drafts (5–7 over ~5 days) scheduled through Hundrads to land BEFORE the ad launch date. Use when the user wants to "warm up" or "seed" a page, says the page looks dead/empty, or is about to run ads on a fresh account. The user approves each draft on the Schedule tab; nothing posts without their Approve.
---

# Seed a page through Hundrads (pre-ads warm-up)

When ads start running, people who click check the profile. An empty or dead
page kills the conversion the ad paid for. This skill schedules a quick burst
of **organic-looking** drafts so that by ad-launch day the page reads like a
real, active account — not a shell that exists to run ads.

Seeds are the opposite of ads: no CTA, no price, no link-pushing. They exist
to pass the 5-second "is this account real?" check.

> **Safety model.** You cannot post anywhere. Drafts go to the Hundrads
> Schedule tab; only the user's explicit Approve arms each auto-post. You
> cannot approve, edit, or un-reject drafts yourself. Don't ask for platform
> credentials.

## Setup

`HUNDRADS_API_KEY` env var. API base URL: `https://hundrads.com`. Endpoint reference: `https://hundrads.com/llms.txt` (markdown API reference).

**Channel reality:** Hundrads auto-posts **Threads** and **Telegram** content
drafts only. There is no Facebook-page or IG-feed sender. If the surface that
needs seeding is a FB page / IG profile, you can still draft those posts —
deliver them in chat as a dated checklist the user posts manually — and say
so plainly up front.

This skill pairs with [`meta-ads`](../meta-ads/SKILL.md): seed first, then
draft the ads; ads launch after the last seed lands.

## Step 1 — Quick intake (a sprint, not a launch)

Ask only what you can't infer from what the user shares:

- **When do ads go live?** Everything schedules BEFORE that date. The only
  hard input.
- **Which surface needs to look alive?** Where does the ad's profile click
  actually land — Threads, Telegram, FB page, IG?
- **How many posts?** Default **5–7 over ~5 days** (minimum 3 days). Ads
  launch tomorrow → compress: 3 seeds today beats 0.
- What is the page about / what will the ads sell? One sentence — seeds
  orbit the topic, they don't sell it.
- Real material to work with — things built, results, opinions, daily
  moments? Real beats invented; never fabricate results or testimonials.

Propose the burst plan in one message (post list + slots) and get a single
OK — no heavy calendar ceremony.

## Step 2 — Write seeds that don't smell like marketing

A seeded page fails if every post reads like a brand warming up to sell.
Mix archetypes — roughly one each, adjusted to count:

0. **Read the brand brief first** (skip only if the user's request is
   self-contained):

   ```bash
   curl -s "https://hundrads.com/v1/brand/brief?brand=<brand>" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Brands come from `GET /v1/accounts`. The brief carries the user's voice
   notes and channel playbooks — follow them over any generic style. Also
   study past posts in the library endpoints to match what actually performed.

1. **Identity / intro** — who this account is, casually. A person talking,
   not a mission statement.
2. **Value drop** — one genuinely useful tip in the niche, complete in
   itself, no "follow for more".
3. **Behind-the-scenes** — what's being built right now. (Plants context for
   the product the ads will sell — without naming the offer.)
4. **Relatable take** — an opinion or moment people in the niche nod at.
5. **Proof of life** — a real result or small win. Only if real material
   exists.
6–7. Second value drop / second take, different angle.

Rules:
- **Zero selling.** No price, no "DM me", no CTA. The ads do the selling
  later.
- Channel voice still applies — Threads seeds follow
  [`threads-post`](../threads-post/SKILL.md) rules (≤500 chars, blunt, no
  engagement bait); Telegram seeds follow
  [`telegram-post`](../telegram-post/SKILL.md) (broadcast voice, ≤4096).
  Study whatever past posts the user shares and match.
- Vary length and type. Five same-shaped posts in a row reads botlike.

Show all seeds in chat; iterate until the user OKs. Never submit copy the
user hasn't seen.

## Step 3 — Schedule like a human, not a cron job

```bash
curl -s "https://hundrads.com/v1/schedule" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

- Don't collide with booked slots.
- **Irregular times.** Not 09:00 five days straight — 21:40, 13:15, 08:55.
  Same-time daily posting is the #1 "this page is automated" tell.
- 1–2 posts/day max per channel; everything lands ≥ a few hours before the
  ad launch slot.
- Telegram seeds may carry `payload.image_url` — the user's real
  photos/screenshots beat generated images (obviously-AI images on a seeded
  page defeat the purpose). Threads seeds go text-only; note any
  manual-media pairing in `agent_note`.

## Step 4 — Submit

One `POST /v1/drafts` per seed:

```bash
curl -s -X POST "https://hundrads.com/v1/drafts" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "kind": "threads_post",
    "agent_note": "Page seed 2/6 ahead of ads launching 2026-06-20 — value drop archetype. Organic tone, no CTA by design.",
    "payload": {
      "text": "<the seed, ≤500 chars>",
      "chain": [],
      "account": "<the handle>",
      "scheduled_at": "2026-06-16T21:40"
    }
  }'
```

(`telegram_post` payload per [`API.md`](https://hundrads.com/llms.txt).) Number every
`agent_note` (`seed N/M` + archetype + the ad launch date) so the user sees
the plan on the Schedule tab.

FB-page / IG seeds (no sender): deliver in chat as a dated checklist for
manual posting and say exactly that in the summary.

Relay all `warnings` + `review_url`, then stop. Natural next step: once
seeds are approved, run [`meta-ads`](../meta-ads/SKILL.md) — ads launch
after the last seed lands.

## Etiquette

- Rejected ≠ retry: `GET /v1/drafts?status=rejected` → read `review_note`,
  change approach, resubmit into the original slot.
- Never claim something was posted — only the dashboard decides.
- Never fabricate proof, results, or testimonials in seeds.
- Never ask for or handle platform credentials. Hundrads is the only door.

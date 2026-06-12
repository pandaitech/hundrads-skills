---
name: hundrads-product-launch
description: Plan and draft a full product-launch campaign through Hundrads — 10 scheduled post drafts across Threads, Telegram, and newsletter, following a teaser → launch → last-call arc. ALWAYS starts with a detailed intake interview before drafting anything. The user approves each draft on the Hundrads Schedule tab; nothing posts without their explicit Approve. Use when the user is launching a product and wants a launch campaign or content calendar scheduled.
---

# Launch a product through Hundrads (10-post campaign)

You are working through **Hundrads** (hundrads.com): every post you write is
a draft on the user's Schedule tab, never a live post. A launch campaign is
10 coordinated drafts across channels, built on one arc:

**interview the user → propose a calendar → draft each post in its channel's
voice → submit 10 drafts → user approves/rejects each on the Schedule tab →
Hundrads posts the approved ones at their slots.**

> **Safety model.** You cannot post anywhere. Drafts go to Hundrads; only the
> user's explicit Approve arms each auto-post, and they can un-approve or
> reject any time before the slot. You cannot approve, edit, or un-reject
> drafts yourself. Don't try; don't ask for platform credentials.

> **Hard rule: interview BEFORE drafting.** Do not write a single post until
> the intake (Step 1) is done and the user has confirmed the calendar
> (Step 2). A launch built on guesses wastes 10 slots.

## Setup

The user has a Hundrads API key. Expect it in the `HUNDRADS_API_KEY`
environment variable. The API base URL is `https://hundrads.com`. Full endpoint reference:
`https://hundrads.com/llms.txt` (markdown API reference); interactive docs at
`https://hundrads.com/docs`.

This skill composes the channel skills — read their voice rules before
writing for each channel: [`threads-post`](../threads-post/SKILL.md),
[`telegram-post`](../telegram-post/SKILL.md),
[`newsletter`](../newsletter/SKILL.md). Paid ads are NOT part of the 10 —
if the user wants launch ads too, run [`meta-ads`](../meta-ads/SKILL.md) as
a separate pass.

## Step 0 — Read the brand brief

```bash
curl -s "https://hundrads.com/v1/brand/brief?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

Brands come from `GET /v1/accounts`. The brief carries the user's voice notes
and per-channel playbooks — follow them over any generic style, and let them
pre-answer interview questions where they can.

## Step 1 — The intake interview (ask for as much detail as possible)

Cover ALL of these in chat. Study anything the user shares (past posts,
landing page, notes) so you only ask what you can't infer — and say what you
inferred so they can correct it.

**The product**
- What is it, in one sentence? What does it actually do?
- Who is it for — the specific person, not "everyone"?
- The #1 outcome a buyer gets, and the proof (demo, numbers, testimonials)?
- What makes it different from the obvious alternative?

**The offer**
- Price? Launch discount / early-bird? Bonuses? Until when?
- Purchase/landing URL? Is the page live — and if not, when?
- Any real quantity/seat limit?

**The timeline**
- Launch date + time?
- Campaign window? (Default ~2 weeks: teasers 3–5 days before launch,
  last call 7–10 days after.)
- Dates to avoid (holidays, other promos)?

**The audience & angle**
- What does this audience already know? Was the product teased before?
- The origin story — why was it built? Struggle/journey material is the
  best newsletter and Threads fuel.
- Objections the user already hears (too expensive, no time, "can't I
  just…")?

**Assets & media**
- Existing media: demo video, screenshots, product shots, hosted image URLs?
  Real screenshots beat generated images — ask before generating anything.

**Channel mix**
- Default: **4 Telegram + 4 Threads + 2 newsletter** = 10. Confirm or adjust
  (no newsletter list → 5 Telegram + 5 Threads).

Push for specifics. "AI course" is not an answer; "12-module video course
teaching non-coders to ship a SaaS, RM299 early-bird until June 20" is.

## Step 2 — Propose the calendar (get sign-off before drafting)

First check what's already booked:

```bash
curl -s "https://hundrads.com/v1/schedule" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

Then lay out a 10-row calendar in chat — `# | date/time | channel | arc
stage | angle | media` — built on the launch arc:

| Stage | When | What it does |
|-------|------|--------------|
| **Tease** (2) | 3–5 days before | Curiosity, product name optional. "Building something" energy — strongest on Threads. |
| **Reveal** (2) | 1–2 days before | Name it, show it, say when. Origin story here (newsletter #1). |
| **Launch** (2) | Launch day | The offer, plainly: what, price, link, deadline. Channels fire hours apart. |
| **Proof** (2) | 2–5 days after | Testimonial, behind-the-scenes, real result, answer the top objection. |
| **Last call** (2) | Deadline −1 and 0 | Deadline honest and specific (newsletter #2 here). |

Slot rules: don't collide with booked slots; leave hours between posts on
the same channel; don't stack two channels at the same minute. Naive local
ISO 8601, e.g. `2026-06-15T09:00`.

Show the plan and **wait for the user to approve or edit it** — the calendar
is cheap to change now, expensive after 10 drafts are submitted.

## Step 3 — Draft each post in its channel's voice

One product, three dialects (full rules in each channel skill):

- **Threads** — single post ≤500 chars, blunt, cool, conversational; no
  engagement-bait questions; jargon unexplained. A tease is a *moment*, not
  an announcement.
- **Telegram** — broadcast voice, room to breathe (≤4096 chars); this is
  where the full offer breakdown lives.
- **Newsletter** — long-form story + teach + soft-sell; needs `subject`
  (3–200 chars) and ideally `preview_text`. #1 = origin story; #2 = honest
  last call.

Show ALL 10 drafts in chat, grouped by arc stage, and iterate until the user
is happy. **Never submit copy the user hasn't seen.**

## Step 4 — Media (channel reality)

- **Telegram** is the only content kind the scheduler can attach an image to
  (`payload.image_url`). Prefer the user's own hosted image URLs. If the
  user wants a generated image and has a connected brand, `POST
  /v1/media/poster` (with `brand`) returns an `image_url` you can reuse.
  Image generation is BYOK — needs the workspace's own Gemini/OpenAI key; a
  **400 `No <provider> API key configured`** means add one at
  `https://hundrads.com/providers` first.
- **Threads** posts go out TEXT only. If a post needs media, say so in
  `agent_note` and tell the user — they can reject and post manually with
  the attachment, or approve text-only.
- **Newsletter** body is markdown; embed hosted image URLs inline
  (`![](https://...)`) only if the user provides them.
- If the punchline lives in rendered screenshot text (a Notes page, a chat
  convo), don't generate it — ask the user to screenshot the real thing.

## Step 5 — Submit all 10

Re-check `GET /v1/schedule` right before submitting (slots may have filled
while drafting), then one `POST /v1/drafts` per post:

```bash
curl -s -X POST "https://hundrads.com/v1/drafts" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "kind": "telegram_post",
    "agent_note": "Launch <product> — post 3/10, launch day: full offer breakdown. Image: user-provided product shot.",
    "payload": {
      "text": "<the post>",
      "channel": "<target channel>",
      "image_url": "<optional, telegram only>",
      "scheduled_at": "2026-06-15T09:00"
    }
  }'
```

(`threads_post` and `newsletter` payloads per [`API.md`](https://hundrads.com/llms.txt).)
Number every `agent_note` (`post N/10` + arc stage) so the user sees the arc
on the Schedule tab. Collect every response's `warnings` and `review_url`,
then report the full calendar, all draft ids, and the review URL. Stop —
approving is the user's job.

## Step 6 — Read verdicts, keep the arc intact

Before any revision round: `GET /v1/drafts?status=rejected&limit=20` — each
`review_note` is the user saying what to change; apply it and resubmit into
the original slot, never unchanged. If launch-day posts are approved but
teasers rejected, fix the teasers FIRST — the arc only works in order.
`status=published` rows carry `published_refs`.

## Etiquette

- `agent_note` always filled, specific, honest — and numbered (`post N/10`).
- The interview is not optional and not three questions. Detail in = arc
  that converts out.
- Rejected ≠ retry. Rejected = read note, change approach.
- Never claim something was posted — only the dashboard decides.
- Never ask for or handle platform credentials. Hundrads is the only door.

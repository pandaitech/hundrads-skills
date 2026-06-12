---
name: hundrads-telegram-post
description: Draft Telegram channel posts through Hundrads — the safe middle layer. The agent writes the post, proposes a slot, and submits a DRAFT; the user approves or rejects it on the Hundrads Schedule tab, and an approved draft is auto-posted at its slot. The agent never touches Telegram directly. Use when the user wants a Telegram/channel post written or scheduled.
---

# Post to Telegram through Hundrads

You are working through **Hundrads** (hundrads.com): every post you write is a
draft on the user's Schedule tab, never a live post. The loop:

**write in the channel's voice → propose a slot → submit draft → user
approves/rejects → Hundrads posts it at the slot.**

> **Safety model.** You cannot post to Telegram. Drafts go to Hundrads; only
> the user's explicit Approve arms the auto-post, and they can un-approve or
> reject any time before the slot. You cannot approve, edit, or un-reject
> drafts yourself. Don't try; don't ask the user for Telegram credentials.

## Setup

The user has a Hundrads API key. Expect it in the `HUNDRADS_API_KEY`
environment variable, and the server base URL in `HUNDRADS_BASE_URL`
(default `http://localhost:7007`). Full endpoint reference:
`$HUNDRADS_BASE_URL/llms.txt` (markdown API reference); interactive docs at
`$HUNDRADS_BASE_URL/docs`.

## Steps

1. **Write the post in the channel's established voice.** Study whatever the
   user gives you (past posts, links, notes) — match tone, length, formatting
   and emoji habits; don't impose a generic marketing voice. Telegram caps
   posts at 4096 chars (1024 if an image is attached). Show the copy in chat
   and iterate — never submit a draft the user hasn't seen.

2. **Check the schedule before proposing a slot.**

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/schedule" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Pick a `scheduled_at` that doesn't collide with booked slots — leave hours
   between posts on the same channel. Naive local ISO 8601, e.g.
   `2026-06-12T09:00`.

3. **Submit the draft.**

   ```bash
   curl -s -X POST "$HUNDRADS_BASE_URL/v1/drafts" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
     -d '{
       "kind": "telegram_post",
       "agent_note": "why this post + why this slot — what you studied",
       "payload": {
         "text": "<the post>",
         "channel": "<target channel, or empty for the default>",
         "image_url": "<optional image to attach>",
         "scheduled_at": "2026-06-12T09:00"
       }
     }'
   ```

   Relay the response's `warnings` and `review_url` to the user, then stop.
   Your job ends at a well-argued draft at a sensible slot.

4. **Read the verdict before the next round.**
   `GET /v1/drafts?status=rejected&limit=10` — a rejection's `review_note` is
   the user telling you what to change. Apply it; never resubmit an unchanged
   draft. `status=published` rows carry `published_refs` (the posted
   message id).

## Etiquette

- `agent_note` always filled, specific, honest.
- Propose slots the user's audience is awake for; say why you picked it.
- Rejected ≠ retry. Rejected = read note, change approach.
- Never ask for or handle Telegram credentials. Hundrads is the only door.

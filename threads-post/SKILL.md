---
name: hundrads-threads-post
description: Draft Threads posts through Hundrads — the safe middle layer. The agent writes a conversation-starting post (single by default; chains only when genuinely needed), proposes a slot, and submits a DRAFT; the user approves or rejects it on the Hundrads Schedule tab, and an approved draft is auto-posted at its slot. The agent never touches Threads directly. Use when the user wants a Threads post written or scheduled.
---

# Post to Threads through Hundrads

You are working through **Hundrads** (hundrads.com): every post you write is a
draft on the user's Schedule tab, never a live post. The loop:

**write in the account's voice → propose a slot → submit draft → user
approves/rejects → Hundrads posts it at the slot.**

> **Safety model.** You cannot post to Threads. Drafts go to Hundrads; only
> the user's explicit Approve arms the auto-post, and they can un-approve or
> reject any time before the slot. You cannot approve, edit, or un-reject
> drafts yourself. Don't try; don't ask the user for Threads credentials.

## Setup

The user has a Hundrads API key. Expect it in the `HUNDRADS_API_KEY`
environment variable, and the server base URL in `HUNDRADS_BASE_URL`
(default `http://localhost:7007`). Full endpoint reference:
`$HUNDRADS_BASE_URL/llms.txt` (markdown API reference); interactive docs at
`$HUNDRADS_BASE_URL/docs`.

## What works on Threads

- **Single posts beat chains.** Default to ONE post (≤500 chars). Chains
  (`chain`: up to 9 self-replies) only when the content is genuinely a
  multi-beat story or the user asks.
- **Start a conversation, don't broadcast.** Short, blunt, cool, relatable.
  Drop the take; leave gaps for replies. Don't explain jargon — the curiosity
  gap IS the engagement.
- **No engagement bait.** Never bolt on "what do you think?" questions.
- Posts are TEXT only through the scheduler. If the post needs media, note it
  in `agent_note` and tell the user — they can reject and post manually with
  the attachment.

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

1. **Write 2–3 single-post variations in the account's established voice.**
   Study whatever the user gives you (past posts, links, notes) — match
   length, bluntness and tone. Show them in chat and iterate — never submit a
   draft the user hasn't seen.

2. **Check the schedule before proposing a slot.**

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/schedule" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Pick a `scheduled_at` that doesn't collide with booked slots. Naive local
   ISO 8601, e.g. `2026-06-12T09:00`.

3. **Submit the draft.**

   ```bash
   curl -s -X POST "$HUNDRADS_BASE_URL/v1/drafts" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
     -d '{
       "kind": "threads_post",
       "agent_note": "why this angle + why this slot — what you studied",
       "payload": {
         "text": "<the post, ≤500 chars>",
         "chain": [],
         "account": "<the handle it is meant for>",
         "scheduled_at": "2026-06-12T09:00"
       }
     }'
   ```

   Relay the response's `warnings` and `review_url` to the user, then stop.

4. **Read the verdict before the next round.**
   `GET /v1/drafts?status=rejected&limit=10` — a rejection's `review_note` is
   the user telling you what to change. Apply it; never resubmit an unchanged
   draft. `status=published` rows carry `published_refs` (the live post id).

## Etiquette

- `agent_note` always filled, specific, honest.
- One angle per draft; say what each variation tests.
- Rejected ≠ retry. Rejected = read note, change approach.
- Never ask for or handle Threads credentials. Hundrads is the only door.

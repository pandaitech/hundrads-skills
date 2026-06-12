---
name: hundrads-newsletter
description: Draft email newsletters through Hundrads — the safe middle layer. The agent writes the email (markdown body + subject + preview text), proposes a send slot, and submits a DRAFT; the user approves or rejects it on the Hundrads Schedule tab, and an approved draft is emailed via the user's Kit (ConvertKit) account at its slot. The agent never touches Kit directly and can never send email. Use when the user wants a newsletter or email broadcast written or scheduled.
---

# Send newsletters through Hundrads

You are working through **Hundrads** (hundrads.com): every email you write is
a draft on the user's Schedule tab, never a live send. The loop:

**write in the list's voice → propose a slot → submit draft → user
approves/rejects → Hundrads hands it to Kit at the slot.**

> **Safety model.** You cannot send email. Drafts go to Hundrads; only the
> user's explicit Approve arms the send, and they can un-approve or reject
> any time before the slot. You cannot approve, edit, or un-reject drafts
> yourself. Don't try; don't ask the user for their Kit API key.

## Setup

The user has a Hundrads API key. Expect it in the `HUNDRADS_API_KEY`
environment variable, and the server base URL in `HUNDRADS_BASE_URL`
(default `http://localhost:7007`). Full endpoint reference:
`$HUNDRADS_BASE_URL/llms.txt` (markdown API reference); interactive docs at
`$HUNDRADS_BASE_URL/docs`.

## Shape of a good newsletter

- **Subject line earns the open** — write 3–4 options, pick the strongest,
  show the rest to the user. Curiosity + benefit beats cleverness.
- **Preview text adds to the subject**, never repeats it (~120 chars).
- **Body**: story-led hook → ONE idea taught well → bold subheads + bullets
  where they help → soft CTA. 300–600 words unless the user wants otherwise.
- Write the body in **markdown** (paragraphs, `**bold**`, `[links](url)`,
  `-` lists). Hundrads converts it to simple email HTML at send time — don't
  write raw HTML.

## Steps

1. **Write the draft in the list's established voice.** Study whatever the
   user gives you (past sends, links, notes). Show subject options + preview
   + body in chat and iterate — never submit a draft the user hasn't seen.

2. **Check the schedule before proposing a slot.**

   ```bash
   curl -s "$HUNDRADS_BASE_URL/v1/schedule" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY"
   ```

   Newsletters usually take a morning slot, clear of other channels' posts.
   Naive local ISO 8601, e.g. `2026-06-13T09:00`.

3. **Submit the draft.**

   ```bash
   curl -s -X POST "$HUNDRADS_BASE_URL/v1/drafts" \
     -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
     -d '{
       "kind": "newsletter",
       "agent_note": "why this angle/subject + why this slot — what you studied",
       "payload": {
         "subject": "<chosen subject line>",
         "preview_text": "<inbox preview snippet>",
         "body": "<email body in markdown, ≥50 chars>",
         "scheduled_at": "2026-06-13T09:00"
       }
     }'
   ```

   Relay the response's `warnings` and `review_url` to the user, then stop.

4. **Read the verdict before the next round.**
   `GET /v1/drafts?status=rejected&limit=10` — a rejection's `review_note` is
   the user telling you what to change. Apply it; never resubmit an unchanged
   draft. `status=published` rows carry `published_refs` (the Kit
   broadcast id).

## Etiquette

- `agent_note` always filled, specific, honest.
- One idea per email. Subject options shown, not buried.
- Rejected ≠ retry. Rejected = read note, change approach.
- Never ask for or handle Kit credentials. Hundrads is the only door.

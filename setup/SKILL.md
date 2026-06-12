---
name: hundrads-setup
description: Connect this agent to the user's Hundrads workspace (hundrads.com — the safe middle layer between AI agents and marketing accounts). Runs a one-time browser device-authorization flow, stores the issued API key in .env, and verifies the connection. Use when the user says "set up hundrads", "connect hundrads", "hundrads init/login", when another hundrads-* skill fails with 401, or right after the hundrads skills were installed.
---

# Hundrads setup — connect this agent

One-time handshake. You (the agent) request a device code, the user
approves it in their browser, Hundrads issues an API key scoped to the
user's workspace, and you store it. No password ever touches the agent.

The key lets this agent **create drafts and read data only** — nothing
posts or spends without the user clicking Approve in the Hundrads
dashboard. The user can revoke this key any time on the Keys tab.

## 0. Already connected?

```bash
[ -f .env ] && export $(grep -E "^HUNDRADS_(API_KEY|BASE_URL)=" .env | xargs)
BASE="${HUNDRADS_BASE_URL:-https://hundrads.com}"
curl -s -o /dev/null -w "%{http_code}" "$BASE/v1/schedule" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

`200` → already connected; tell the user and stop. Anything else → continue.

## 1. Request a device code

```bash
curl -s -X POST "$BASE/device/code" \
  -H "content-type: application/json" \
  -d '{"client_name": "<agent name> @ <machine/project name>"}'
```

Pick a `client_name` the user will recognize on their Keys tab (e.g.
`"Claude Code @ macbook"`). Response:

```json
{"device_code": "dvc_…", "user_code": "ABCD-1234",
 "verification_url": "https://hundrads.com/device",
 "expires_in": 900, "interval": 5}
```

## 2. Send the user to approve

Show the user — prominently, then WAIT:

> Open **<verification_url>?code=<user_code>** and click **Approve**.
> Code: **<user_code>** (expires in 15 minutes)

If you can open a browser for the user, open that URL.

## 3. Poll until approved

Every `interval` seconds (never faster):

```bash
curl -s -X POST "$BASE/device/token" \
  -H "content-type: application/json" \
  -d '{"device_code": "<device_code>"}'
```

- `{"status": "pending"}` → keep polling.
- `{"status": "ok", "api_key": "hnd_live_…"}` → success. The key is
  returned ONCE — store it immediately (step 4).
- `{"status": "denied"}` → the user said no. Stop; ask what they'd like.
- `{"status": "expired"}` → restart from step 1.

## 4. Store the key

Write both values into `.env` in the project root (create it if missing;
replace existing lines for these two vars, never duplicate):

```
HUNDRADS_BASE_URL=https://hundrads.com
HUNDRADS_API_KEY=hnd_live_…
```

Never print the full key in chat or logs after this point. If `.env` is
not gitignored, warn the user and add it to `.gitignore`.

## 5. Verify + hand off

```bash
curl -s "$BASE/v1/accounts" -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

On 200, tell the user:
- Connected to their Hundrads workspace; key revocable at `$BASE/keys`.
- Which brands/accounts came back (from the response).
- What they can ask for now: ad drafts, Telegram/Threads posts,
  newsletters, launch campaigns — every draft lands in the Hundrads
  dashboard for THEIR approval; nothing posts or spends on its own.

## Troubleshooting

- `401` on /v1 after setup → key revoked or .env not loaded into the
  shell; re-run this skill.
- `Unknown or expired code` in the browser → the 15 minutes passed;
  restart from step 1.
- Self-hosted server → ask the user for their base URL and use it
  instead of https://hundrads.com everywhere above.

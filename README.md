# Hundrads skills

Skills that connect your AI agent to [Hundrads](https://hundrads.com) —
the safe middle layer between agents and your marketing accounts.

**The contract:** agents draft, you approve. Every ad, post, and
newsletter your agent writes lands in the Hundrads dashboard as a draft.
Nothing posts, sends, or spends without your explicit Approve — and even
approved Meta ads are created PAUSED.

## Install

```bash
npx skills add pandaitech/hundrads-skills
```

Then tell your agent: **"set up hundrads"** — the `setup` skill walks it
through a one-time browser approval that issues your API key (revocable
any time at [hundrads.com/keys](https://hundrads.com/keys)).

## Skills

| Skill | What your agent can do with it |
|---|---|
| `setup` | One-time connect: device auth → API key → verified |
| `meta-ads` | Draft Meta (FB/IG) ads learned from your own past ad performance |
| `manage-ads` | Daily marketer loop on live ads — kill/scale/fix proposals, never direct changes |
| `meta-ads-autopilot` | Hands-off autopilot: runs the marketer loop on a schedule, drafts changes + new ads, reports to your channel |
| `telegram-post` | Draft Telegram channel posts in your voice, scheduled |
| `threads-post` | Draft Threads posts in your voice, scheduled |
| `newsletter` | Draft email newsletters (Kit) in your voice, scheduled |
| `product-launch` | Full 10-post launch campaign across channels |
| `seed-page` | Warm up a page with organic-looking posts before ads |
| `pfp-banner` | Generate page pfp + cover/banner images (Nano Banana) — you upload them |

API reference: [hundrads.com/llms.txt](https://hundrads.com/llms.txt) ·
interactive docs: [hundrads.com/docs](https://hundrads.com/docs)

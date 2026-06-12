---
name: hundrads-pfp-banner
description: Generate social media profile pictures and banners (mainly Facebook — page pfp + cover photo) with Nano Banana (Gemini image) through the Hundrads media API. Use when the user wants a pfp, avatar, profile picture, banner, cover photo, or page header image for FB/IG/Threads/X. Delivers ready-to-upload PNG files; the user uploads them to the platform themselves — this skill never touches any account.
---

# Page identity assets: pfp + banner (Nano Banana)

A page that runs ads needs to look like a real brand. The two assets every
profile click hits first: the profile picture and the cover/banner. This
skill generates both with Nano Banana (Gemini's image model) via
`POST /v1/media/poster` and hands the user finished PNG files.

> **Safety model.** This skill only generates images. There is no endpoint
> that sets a profile picture or cover photo anywhere — the user downloads
> the files and uploads them in the platform's own UI. Don't ask for
> platform credentials.

## Setup

`HUNDRADS_API_KEY` env var. API base URL: `https://hundrads.com`. Endpoint reference: `https://hundrads.com/llms.txt`.

## Step 1 — Quick intake

Ask only what you can't infer:

- **Which brand / page?** Brands come from `GET /v1/accounts`.
- **Which platform(s)?** Default Facebook page (pfp + cover). Same pfp
  works for IG/Threads/X; banners differ per platform (see specs below).
- **Text on the assets?** Brand name on the pfp? Tagline on the banner?
  Get the exact spelling — generated text must match character-for-character.
- **Style direction**, if the brief doesn't already carry one (colors,
  mood, logo-like vs photo-like).

Then read the brand brief — it carries voice, colors, and positioning:

```bash
curl -s "https://hundrads.com/v1/brand/brief?brand=<brand>" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY"
```

### Scan the working folder for an existing logo / brand reference

Before writing any prompt, look for a real logo or brand asset already
sitting in the folder this skill runs in — most brands have one. Don't
invent a logo when one exists.

```bash
find . -maxdepth 3 \
  \( -iname '*logo*' -o -iname '*brand*' -o -iname '*icon*' -o -iname '*mark*' \
     -o -iname '*wordmark*' -o -iname '*favicon*' \) \
  \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.svg' -o -iname '*.webp' \) \
  -not -path '*/node_modules/*' 2>/dev/null
```

Also glance at the `brand-assets/` output dir and any `assets/`, `public/`,
`static/`, or `img/` folders for unobviously-named brand imagery.

If a logo is found:

1. **Read it** (open the image, or for `.svg` read the markup) so you can
   describe it precisely — shape, colors (hex), letterforms, icon motif.
2. The poster endpoint takes a **text prompt only** — no image upload. So
   feed the logo into the prompt as an exact verbal description: e.g.
   *"the brand's existing logo: a rounded teal hexagon (#0FB5AE) enclosing a
   white lowercase 'h', flat, no gradient"*. The closer the description, the
   closer the generated asset matches the real mark.
3. If the logo carries text you must reproduce exactly, quote it (same rule
   as any generated text — character-for-character).

If nothing is found, don't fabricate one — ask the user for the real logo,
or generate text/logo-free and let them composite (see Etiquette).

## Step 2 — Design for the crop, not the canvas

Platform display realities the prompt must account for:

| Asset | Generate | Displayed as | Design rule |
|---|---|---|---|
| FB/IG/Threads/X pfp | `1:1` | Circle, ~170px desktop / ~40px in feed | One bold central element, fills ~70% of frame, nothing important in corners (circle crop eats them), readable as a thumbnail |
| FB cover | `16:9` | ~820×312 desktop, ~640×360 mobile — top/bottom cropped | All text + key elements in the central horizontal band (middle ~50% of height) and central ~75% of width |

Prompt construction:

- **Pfp**: flat, logo-like, high contrast, single subject or monogram,
  solid/simple background from the brand palette. Avoid busy scenes — at
  40px they turn to mud.
- **Banner**: wide composition, brand colors, tagline (if any) centered in
  the safe band; explicitly tell the model to keep generous empty margin at
  top and bottom edges.
- Include exact brand colors (hex if known), the exact text in quotes and
  its language, and "no watermark, no lorem ipsum, no extra text".

## Step 3 — Generate

`complexity: "simple"` routes to Nano Banana (the Gemini image model in the
registry) — cheap, good for clean graphic compositions. Every call is
logged to the workspace Logs tab automatically.

```bash
curl -s -X POST "https://hundrads.com/v1/media/poster" \
  -H "Authorization: Bearer $HUNDRADS_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "prompt": "<full visual description, exact text in quotes>",
    "complexity": "simple",
    "aspect": "1:1",
    "include_b64": true
  }' > /tmp/pfp.json

python3 -c 'import json,base64; d=json.load(open("/tmp/pfp.json")); open("brand-assets/<brand>-pfp.png","wb").write(base64.b64decode(d["png_b64"]))'
```

Repeat with `"aspect": "16:9"` for the banner →
`brand-assets/<brand>-cover.png`.

Rules:

- **Never print `png_b64`** to chat or read the JSON into context — it's
  hundreds of KB. Always pipe the response to a file and decode from there.
- Leave `brand` empty — that field uploads to a Meta **ad account**
  (`image_hash` for ad drafts), which is not what a pfp/cover needs.
- Generated text wrong/misspelled? Retry once with the text quoted and
  emphasized in the prompt. Still wrong after 2 tries → switch that asset to
  `"complexity": "complex"` (the accurate-text model; costs more — say so),
  or generate it text-free and tell the user to overlay text themselves.

## Step 4 — Deliver + iterate

1. Show the user both images (send the files, not base64).
2. Iterate on feedback — vary one thing per regeneration (color, subject,
   text placement) so the user can steer.
3. When approved, deliver the files with upload instructions:
   - **FB page**: Page → Edit → profile picture / cover photo. FB will
     circle-crop the pfp and crop the cover top/bottom — the safe-band
     design above means nothing important gets cut.
   - Same pfp file works on IG, Threads, X. For non-FB banners (X 3:1,
     LinkedIn 4:1), the 16:9 cover usually crops fine if the safe band held;
     offer a re-generate per platform if it didn't.

## Etiquette

- Never claim anything was uploaded — the user does every upload.
- Never invent a logo for a brand that already has one; ask for the real
  one and describe it in the prompt (or generate text-free and let the user
  composite).
- Pfp + cover should read as one identity: same palette, same mood.
  Generate the pfp first, then describe it in the banner prompt.

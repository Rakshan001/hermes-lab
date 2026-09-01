# Demo runbook — Telegram → generate → auto-post to Instagram

**Written:** 2026-09-01 · Sandbox only. Read `LOCAL-SETUP-CONTEXT.md` first.

Goal: message the bot on Telegram, it writes a caption, generates an image, posts to a
dummy Instagram account, and a cron job pulls the post's stats back.

---

## 1. The key checklist

| # | Key | For | Cost | Required for demo? |
|---|---|---|---|---|
| 1 | `ANTHROPIC_API_KEY` | Text — captions, suggestions | Paid, you have it | **Yes** |
| 2 | `FAL_KEY` | Images + video | Paid, free signup credits | **Yes** — Hermes has no other image backend |
| 3 | Meta app + IG token | Instagram publish + insights | **Free** | **Yes** |
| 4 | `TELEGRAM_BOT_TOKEN` | Chat surface | **Free** | Already set |
| 5 | `OPENAI_API_KEY` | Alternative text/image | Paid, you have it | Optional |
| 6 | `GEMINI_API_KEY` | Alternative text/image | Paid, you have it | Optional |
| 7 | `PERPLEXITY_API_KEY` | Competitor research | Paid, you have it | Optional — nice extra |

**Three keys to actually get: `FAL_KEY`, and the Meta app token. Anthropic you have.**

---

## 2. fal.ai — get the key

1. https://fal.ai → sign up
2. Dashboard → **Keys** → create key
3. New accounts get **free signup credits** (amount varies by promotion) — enough to
   test without paying
4. `echo 'FAL_KEY=...' >> ~/.hermes/.env` (keep mode 600)
5. Enable the toolset:

```yaml
# ~/.hermes/config.yaml
platform_toolsets:
  telegram: [clarify, code_execution, delegation, file, memory,
             session_search, skills, terminal, todo, image_gen]
```

6. Restart the gateway, then over Telegram: *"generate an image of a coffee cup on a
   wooden table, warm morning light"*

### Why fal solves a problem you would otherwise have to solve

**Instagram will not accept uploaded image bytes. It fetches a public HTTPS URL.**

fal returns generated images at hosted public URLs. That means step 4 of the demo — "put
the image somewhere public" — is already done. Without fal you would need an S3 bucket
with public objects, or an ngrok tunnel, before you could post anything.

For a demo this is the single biggest practical advantage.

---

## 3. Instagram — full setup

Free. No payment anywhere in this section.

### 3.1 Prepare the accounts

1. In the Instagram app on the dummy account:
   **Settings → Account type and tools → Switch to professional account → Business**
2. Create a Facebook Page (or use an existing one)
3. Link them: **Instagram → Settings → Account Centre**, or from the Page's settings.
   *The Page link is not optional — the API reaches Instagram through the Page.*

### 3.2 Create the Meta app

1. https://developers.facebook.com → **My Apps → Create App**
2. Type: **Business**
3. Add the product: **Instagram Graph API** (and **Facebook Login** for token flow)

### 3.3 Permissions

```
instagram_basic
instagram_content_publish      ← the one that matters
pages_show_list
pages_read_engagement
instagram_manage_insights      ← for the cron stats fetch
```

**In Development mode, these work on accounts that hold a role on the app** (admin,
developer, or tester). Your dummy account qualifies. **App Review is only needed to
publish to other people's accounts** — a phase-3 concern for the real product, not for
this demo.

### 3.4 Get a token

1. **Tools → Graph API Explorer**
2. Select your app, add the permissions above, **Generate Access Token**
3. That token is short-lived (~1 hour). Exchange it for a long-lived one (**60 days**):

```
GET https://graph.facebook.com/v21.0/oauth/access_token
  ?grant_type=fb_exchange_token
  &client_id={app-id}
  &client_secret={app-secret}
  &fb_exchange_token={short-lived-token}
```

**Set a calendar reminder for day 55.** A silently expired token presents as "the demo
stopped working" at the worst possible moment. In the product this is an automated
refresh job (`social_connections.expires_at`, health `EXPIRING`).

### 3.5 Find the Instagram Business Account ID

```
GET /me/accounts                          → page-id
GET /{page-id}?fields=instagram_business_account   → ig-user-id
```

Keep `ig-user-id`. Every publish call uses it.

### 3.6 Publish — always two calls

```
① CREATE THE CONTAINER
POST https://graph.facebook.com/v21.0/{ig-user-id}/media
  ?image_url={PUBLIC_HTTPS_URL}          ← fal's hosted URL
  &caption={URL-encoded caption}
  &access_token={token}
→ { "id": "<creation-id>" }

② PUBLISH IT
POST https://graph.facebook.com/v21.0/{ig-user-id}/media_publish
  ?creation_id={creation-id}
  &access_token={token}
→ { "id": "<media-id>" }
```

Notes that save an hour of debugging:

- The URL must be **publicly reachable HTTPS**. `localhost` fails. A signed S3 URL with
  a short expiry can fail if Meta fetches late.
- JPEG/PNG, aspect ratio between 4:5 and 1.91:1. A 9:16 image is **rejected** for a
  feed post.
- **Rate limit: 25 posts per rolling 24 hours** per account. Ample for a demo; worth
  knowing before a bulk test.
- Container creation is async for video — poll `status_code` until `FINISHED` before
  publishing.

### 3.7 Cron — fetch post stats back

Per post:

```
GET /{ig-media-id}/insights
  ?metric=impressions,reach,likes,comments,saved,shares
  &access_token={token}
```

Account level:

```
GET /{ig-user-id}/insights?metric=impressions,reach,profile_views&period=day
```

Hermes has a `cronjob` toolset already enabled for CLI. Schedule the fetch, write the
results somewhere, and report over Telegram.

**In the product** this feeds the `outcome` memory (§05, source 3) — "the sustainability
angle got 3× the reach of the discount angle" is exactly what makes week four's
suggestions better than week one's.

---

## 4. Cost — with a correction

I previously said direct provider keys are always cheaper than fal because fal adds a
margin. **Checking actual prices, that is wrong for at least one model.**

### Image, per 1024×1024 (list prices, Sept 2026 — verify, these move)

| Route | Model | Price |
|---|---|---|
| **fal** | Seedream V4 | **$0.03** |
| **fal** | Flux Schnell | **$0.025** |
| **fal** | Flux Kontext Pro | **$0.04** |
| **fal** | Nano Banana | **$0.0398 / MP** |
| **direct Google** | Gemini 3.1 Flash Image (Nano Banana) | **$0.067** ← *more than fal* |
| **direct Google** | Gemini 3.1 Flash Lite Image | $0.0336 |
| **direct Google** | Imagen 4 Fast / Standard / Ultra | $0.02 / $0.04 / $0.06 |
| **direct OpenAI** | GPT Image 1.5 | $0.009–$0.20 by quality |
| **direct OpenAI** | GPT Image 1-mini | from ~$0.005 |

**The honest conclusion: it depends on the model and quality tier, and the naive
assumption is wrong.**

- **fal's Nano Banana ($0.0398/MP) is cheaper than Google's own list price ($0.067).**
  fal buys at volume and can undercut list.
- **OpenAI's low-quality and mini tiers are far cheaper than anything on fal**
  (~$0.005–0.009).
- Imagen 4 Fast direct at $0.02 beats most fal options.

So: **measure on your actual briefs, do not assume.** This is exactly why the model
gateway in `hermes-studio/docs/plans/17` exists — the routing table makes this a config
decision you can revisit monthly.

### The lever that beats all of the above

**Batch APIs from both OpenAI and Google cut cost by ~50%.**

Your suggestion and image generation runs on **cron**, not interactively. Cron work is
the textbook batch case — nobody is waiting. Routing scheduled generation through batch
endpoints is a larger saving than any provider choice on this page, and it costs you
nothing in quality.

Interactive work (a user clicking "regenerate") stays on the synchronous path.

### Free credits

fal gives **free credits on signup**; the amount varies by promotion. Purchased credits
expire after 365 days, promotional credits after 90. Enough to run this demo without
paying anything.

Sources — check current prices before committing:
[fal pricing](https://fal.ai/pricing) ·
[OpenAI API pricing](https://developers.openai.com/api/docs/pricing) ·
[Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing)

---

## 5. How safe is a fal API key?

**Honest assessment: it is a spending credential with no scoping.**

| Risk | Reality |
|---|---|
| Leaked key | Someone generates images on your account until credits run out |
| Blast radius | **Money only.** No customer data, no social accounts, no publishing rights |
| Scoping | None — one key, full account access |
| Recovery | Rotate the key. Losses are capped by remaining credit |

Compare that with what else is lying around: a leaked `ANTHROPIC_API_KEY` is also money.
A leaked **Instagram token** lets someone post to the account. A leaked **KMS grant**
in the real product would expose every client's social tokens. On that scale a fal key
is the *least* dangerous credential in the stack.

**Handling rules:**

- Server-side only. Never in a browser, never in client JS, never in a repo.
- In the sandbox it lives in `~/.hermes/.env` at mode 600. The Docker terminal cannot
  read it — `docker_forward_env` is empty, which is why that setting matters.
- Set a spending cap in the fal dashboard if one is offered. Prepaid credit is itself a
  cap.
- Rotate after the demo if the key was ever shared or pasted anywhere.
- **In the product:** Secrets Manager, and only the image worker's IAM role can read it.
  The engine sandbox never sees it.

---

## 6. What fal buys you — beyond generation

| Advantage | Why it matters here |
|---|---|
| **Public hosted output URLs** | **Instagram requires a public HTTPS URL. This alone removes a whole build step.** |
| One key → many models | flux, gpt-image, nano-banana, veo, z-image — including FLUX, which needs Black Forest Labs or Replicate to reach directly |
| **`/edit` endpoints on every model** | Inpainting = the revision loop. "Warmer background" without regenerating |
| Native Hermes support | Zero integration work. The tool already exists |
| Queue handling | Long video jobs do not block |
| Video | `fal-ai/veo3.1/image-to-video` — Veo 3.1, hard to reach otherwise |

**For the demo, fal is the right choice, and not mainly because of price.** It is the
only backend Hermes speaks, and it hands you hosted URLs that Instagram can fetch.

**For the product, keep it as one route among several** behind your own gateway —
primary for FLUX and video, with direct OpenAI/Google for volume image work on batch
endpoints.

---

## 7. Order of work

```
□ 1. Anthropic key into Hermes  (provider: anthropic)     — verify over Telegram
□ 2. fal key + image_gen toolset                          — verify an image returns
□ 3. Instagram Business account + Facebook Page linked
□ 4. Meta app, permissions, long-lived token, ig-user-id
□ 5. Publish by hand with curl                            — prove the two calls work
□ 6. Wire it as a Hermes skill                            — "generate and post about X"
□ 7. Cron job for insights                                — stats back over Telegram
```

**Do steps 5 and 6 in that order.** Getting the two-call publish working with `curl`
first means that when the agent-driven version fails, you know it is the agent and not
Instagram.

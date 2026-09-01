# Should you buy fal.ai credits? — decision report

**2026-09-01** · Short answer: **yes, buy a small amount. But not for the reason you
are asking about.**

---

## First, clear up a misconception

> "or gemini key separate subscriptions I have to pay right?"

**No. There are no subscriptions here.** Every option is pay-as-you-go:

| | Billing |
|---|---|
| OpenAI API key | Metered, billed monthly. **You already have it. Nothing more to buy.** |
| Gemini API key | Metered. **You already have it. Nothing more to buy.** |
| fal.ai | **Prepaid credits** — you buy a balance and draw it down |

So the real question is not "which subscription", it is **prepaid credit vs metered
billing on accounts you already hold.**

---

## The decision does not turn on price

### Reason 1 — Hermes' image tool speaks fal and nothing else

`tools/image_generation_tool.py`: 25 `fal-ai/` model references, **zero** references to
any other provider. There is no config switch.

**Without `FAL_KEY`, Hermes cannot generate an image at all.** Your OpenAI and Gemini
keys are useless to it. Using them would mean writing your own image tool — a day of
work to save about a dollar on the demo.

### Reason 2 — fal returns public hosted URLs, and Instagram needs one

Instagram does not accept uploaded bytes. It **fetches** an image from a public HTTPS
URL.

fal hands you exactly that. Going direct with OpenAI or Gemini returns base64 or a
short-lived URL, so you would first need a public S3 bucket or an ngrok tunnel before
you could post anything.

**fal deletes a whole build step from the demo.**

### Reason 3 — there is no free alternative

Google's free tier covers the **Gemini app**, not the API. Current pricing pages list
**"Free Tier: Not available"** for the newest image models. There is no free API route
to images.

fal gives **free credits on signup**. It is the only free path you have.

---

## And the price difference is genuinely irrelevant at your volume

| Images / month | fal (Seedream $0.03) | Imagen 4 Fast direct ($0.02) | Direct + batch (−50%) |
|---|---|---|---|
| **100 — your demo** | **$3** | $2 | $1 |
| 1,000 | $30 | $20 | $10 |
| 10,000 | $300 | $200 | $100 |
| 50,000 — at real scale | $1,500 | $1,000 | **$500** |

**At demo volume the difference is one dollar.** Optimising it costs a day of
engineering, which is worth more than a decade of the saving.

**The crossover is roughly 10,000 images/month.** Below that, use whatever is fastest to
build. Above it, the difference becomes $1,000/month and is worth routing properly.

You are nowhere near that, and will not be for months.

---

## What to buy

**$10–20 of credits.** That is 300–600 images at typical prices — far more than a demo
needs, and it will likely arrive on top of free signup credits.

Do **not** buy a large balance. Credits expire: **365 days for purchased, 90 days for
promotional.** Buying $200 "to be safe" is buying something you may not use before it
expires.

---

## What fal actually gives you

| Feature | Why it matters here |
|---|---|
| **Public hosted output URLs** | Instagram requires it. Removes a build step |
| One key → many models | flux, gpt-image, nano-banana, z-image, veo |
| **`/edit` endpoints on every model** | Inpainting = the revision loop your product needs |
| Native Hermes support | Zero integration work — the tool already exists |
| Queue handling | Long video jobs do not block |
| **FLUX access** | Otherwise needs Black Forest Labs or Replicate directly |
| **Veo 3.1 video** | Hard to reach any other way |

**FLUX and Veo are the two you genuinely cannot get cheaply elsewhere.** Everything
else is convenience — but convenience is exactly what a demo is optimising for.

---

## The honest downsides

| | |
|---|---|
| Another vendor dependency | Real, but it is one key and one HTTP call |
| Credits expire | 365 days purchased / 90 promotional. Buy small |
| No per-project key scoping | One key, full account access |
| A margin on resold models | Sometimes. It also **undercuts** Google on Nano Banana ($0.0398/MP vs $0.067) |
| fal-specific code is throwaway if you later go direct | **Not applicable** — Hermes' tool is already written. You write nothing |

That last row matters: the usual argument against an aggregator is lock-in through
integration code. Here you are writing none, so there is nothing to throw away.

---

## Verdict

**Buy $10–20 of fal credits today.**

Not because it is cheaper — at scale it often is not. Because:

1. It is the **only** image backend Hermes speaks
2. It solves Instagram's public-URL requirement for free
3. There is no free alternative
4. At demo volume the price difference is a rounding error

## And for the product, later

Different question, different answer. At 10,000+ images/month, route per task through
your own gateway:

```
cron / batch image work   →  OpenAI or Google direct, BATCH API (−50%)
interactive regeneration  →  whichever wins your bake-off
FLUX + video              →  fal (no good direct route)
```

**The batch API is the biggest lever by far** — your suggestion and image generation run
on cron, where nobody is waiting, and that halves the cost at no quality loss. It is
worth more than any provider choice.

Design that into the routing table (`hermes-studio/docs/plans/17`) from phase 2. Do not
build it now.

---

**Sources** — verify before buying, prices move:
[fal pricing](https://fal.ai/pricing) ·
[OpenAI API pricing](https://developers.openai.com/api/docs/pricing) ·
[Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing) ·
[Gemini free-tier analysis](https://www.aifreeapi.com/en/posts/gemini-image-generation-free-api)

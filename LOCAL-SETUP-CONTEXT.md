# Local Hermes sandbox — setup context

**Written:** 2026-09-01 · **For:** handing this environment to a new session

**This clone is a TESTING SANDBOX. It is not the product.**
The multi-tenant product lives in a separate repo:
`~/Desktop/Developer/work/Product/MIAM-(Marketing)/hermes-studio/`
Nothing here gets shipped. Break it freely.

---

## 1. What is running

| | |
|---|---|
| Hermes | v0.20.5, commit `1a66134404` |
| Install | `~/.hermes/venvs/hermes-dev/` |
| Config | `~/.hermes/config.yaml` |
| Secrets | `~/.hermes/.env` (mode 600) |
| Gateway | running — `python -m hermes_cli.main`, 1 platform (Telegram) |
| Backup | `~/hermes-backup-20260828-121736` (343 MB, `go-rwx`) |

Telegram works. Messages to the bot reach the agent and come back.

---

## 2. Security decisions already made — do not undo these

The install was hardened on 2026-08-28. Each of these was deliberate.

| Removed | Why |
|---|---|
| **cua-driver** | Needed Accessibility + Screen Recording = full control of the Mac |
| **browser-use** | Drove the logged-in Chrome profile — every signed-in session |
| **WhatsApp (Baileys)** | Unofficial WhatsApp Web client; gets numbers banned |
| `computer_use`, `browser` toolsets | Absent from all toolsets — this is the enforcement |

The terminal runs in Docker, verified isolated (container reports `Linux … linuxkit`,
cannot see `/Users`):

```yaml
terminal:
  backend: docker
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: false
  container_persistent: false
  docker_network: true
  container_memory: 4096
  container_cpu: 2
  lifetime_seconds: 300
```

**Two faults were found and fixed** — worth knowing, because both look like "Hermes is broken":

1. `proxy.enabled: true` with no `proxy.yaml` broke **every** terminal command
   ("the Hermes proxy is misconfigured"). Pre-existing — it was in the backup too, and
   almost certainly the original "this is not working". Now `proxy.enabled: false`,
   which is safe because no cloud keys are forwarded into the sandbox
   (`docker_forward_env` is empty).
2. OpenRouter returned **HTTP 402 — out of credits**. Switched to a free model.

---

## 3. Current model — and how to switch to the paid keys

Right now:

```yaml
model:
  default: minimax/minimax-m3:free
  provider: openrouter
  base_url: https://openrouter.ai/api/v1
  api_mode: chat_completions
```

That was chosen only because OpenRouter credits ran out. **There are paid Anthropic,
OpenAI, and Gemini keys available** — Hermes supports all three directly, no OpenRouter
in the path.

From `cli-config.yaml.example:44-47`, the relevant providers are:

```
"anthropic"  → Direct Anthropic API   (requires ANTHROPIC_API_KEY)
"gemini"     → Google AI Studio       (requires GOOGLE_API_KEY or GEMINI_API_KEY)
"openrouter" → OpenRouter             (requires OPENROUTER_API_KEY)
"auto"       → auto-detect from whatever credentials exist
```

**To switch to direct Anthropic:**

```yaml
# ~/.hermes/config.yaml
model:
  default: "anthropic/claude-opus-4.6"
  provider: anthropic
  # delete base_url and api_mode — they are OpenRouter-specific
```

```bash
# ~/.hermes/.env   (keep mode 600)
ANTHROPIC_API_KEY=sk-ant-...
```

Then `hermes gateway restart` (or kill and relaunch) and send a Telegram message to
confirm.

**Gemini instead:** `provider: gemini`, `default: "gemini-3-flash"` (or similar), and
`GEMINI_API_KEY` in `.env`.

Local Ollama also works and was tested — `qwen3:14b-64k` with `num_ctx 65536`, flash
attention on, `OLLAMA_KV_CACHE_TYPE=q8_0`. Correct but slow: ~4m45s per response
versus ~16s hosted. Fine for offline testing, not for demos.

---

## 4. Image generation — it IS built in, just not switched on

A previous session reported it could not generate images and suggested ComfyUI. That
was a wrong turn. **Hermes ships `tools/image_generation_tool.py`**, which routes
through **fal.ai** and already supports:

| Model | Notes |
|---|---|
| `fal-ai/gpt-image-1.5` | plus `/edit` endpoint |
| `fal-ai/nano-banana-pro`, `nano-banana-2` | plus `/edit` |
| `fal-ai/flux-2-pro`, `flux-2/klein/9b` | plus `/edit` |
| `fal-ai/z-image/turbo` | fast/cheap |

There is also `tools/video_generation_tool.py`.

**Every one of those has an `/edit` endpoint — that is inpainting, which is exactly the
revision loop the product needs** ("make the background warmer" without regenerating).

**To enable it:**

```bash
# 1. Get a fal.ai key → https://fal.ai/dashboard/keys
echo 'FAL_KEY=...' >> ~/.hermes/.env

# 2. Add the toolset — toolsets.py:142 defines "image_gen" → ["image_generate"]
```

```yaml
# ~/.hermes/config.yaml
platform_toolsets:
  telegram: [clarify, code_execution, delegation, file, memory,
             session_search, skills, terminal, todo, image_gen]   # ← add
```

Note the Anthropic/OpenAI/Gemini keys do **not** drive image generation here — Hermes
goes through fal. But fal hosts `gpt-image-1.5` and the nano-banana (Gemini) models,
so those models are reachable via a single `FAL_KEY`.

**ComfyUI is the wrong path** for both testing and the product: it needs a GPU, and the
product will call hosted image APIs through its own model gateway.

---

## 5. Next task — Instagram auto-posting over Telegram

The goal: ask the agent over Telegram, have it generate a post and image, and publish
to Instagram.

**There is no shortcut, and the shortcut is dangerous.** Unofficial Instagram posting
libraries get accounts banned. The supported path:

1. Instagram account converted to **Business or Creator**
2. Linked to a **Facebook Page**
3. A Meta app with `instagram_content_publish` and `pages_show_list`
4. Publishing is **two calls**: `POST /{ig-user-id}/media` to create a container, then
   `POST /{ig-user-id}/media_publish` to publish it
5. The image must be at a **public HTTPS URL** — Instagram fetches it. It cannot be
   uploaded as bytes.

That last point shapes the demo: generated images need somewhere public to live, even
if that is a temporary S3 bucket or an ngrok tunnel.

**In development mode, an app can post to accounts you own** without App Review —
enough for a demo. App Review is only needed to publish on *other people's* accounts,
which is a phase-3 concern for the real product.

Suggested build order for the sandbox:
1. Enable `image_gen`, confirm an image comes back over Telegram
2. Get the image to a public URL
3. Wire the two-step Instagram publish as a Hermes skill or a small script
4. Ask over Telegram: *"generate a post about X and publish it"*

---

## 6. Baton — why `npx batonhq@latest setup` failed

```
npm error code ENOVERSIONS
npm error No versions available for batonhq
```

**The package is fine.** `batonhq` exists with 9 published versions, latest `0.2.4`.

The cause is in the warning line above the error:

```
npm warn Unknown project config "min-release-age-exclude"
```

`~/.npmrc` sets **`min-release-age = 14`** — a supply-chain guard that refuses any
package published less than 14 days ago. Every `batonhq` version was published between
**2026-08-21 and 2026-08-27**. Today is 2026-09-01, so the oldest is 11 days old.
**All 9 versions are filtered out → zero eligible → `ENOVERSIONS`.**

`min-release-age` is a good setting. Do not disable it globally.

**And you do not need npx at all — baton is already installed:**

```bash
$ which baton && baton --version
/opt/homebrew/bin/baton
0.2.4                      # ← same as npm latest
```

So just run:

```bash
baton setup
```

If a newer version is ever wanted from npm, add an exclusion rather than removing the
guard:

```bash
npm config set min-release-age-exclude "$(npm config get min-release-age-exclude),batonhq"
```

Baton is already running — the Foxwel app holds connections to `localhost:7077`.

### What baton actually does

It is **worktree orchestration for parallel agents** — `baton new <task>` scaffolds a
branch and worktree, `baton ls` shows the board, `baton status` shows which agent is
live and where conflicts are likely, `baton merge`/`push`/`integrate` land the work.

It is **not** a centralised knowledge base. It coordinates agents working on the same
repo without colliding. Worth being clear about, since that was the expectation — the
knowledge base is a thing the product builds, designed in `hermes-studio/docs/plans/`.

---

## 7. Where the real design lives

```
~/Desktop/Developer/work/Product/MIAM-(Marketing)/hermes-studio/
├── README.md
└── docs/
    ├── plans/00-INDEX.md          ← reading order, 25 documents
    ├── plans/20-build-start-here.md  ← folder structure, user flows, week 1–4
    ├── decisions/                 ← 6 ADRs
    └── reference/                 ← agency-agents + foxwel-studio audits
```

Branch `v2-product-scoped-design`. `main` holds the v1 fallback.

**Key point for anyone picking this up:** the product does **not** fork Hermes. It
vendors it pinned and invokes it statelessly, with memory and session-search tool calls
fulfilled by the application against product-scoped storage. See `ADR-006` and
`docs/plans/21-memory-and-self-learning-port.md`.

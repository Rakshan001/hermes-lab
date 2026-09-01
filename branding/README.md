# Foxwel Studio white-labelling

How the assistant is made to answer as Foxwel Studio rather than as the
underlying engine.

> **This is branding, not a security boundary.** A prompt shapes behaviour; it
> does not enforce it. A determined prompt-injection attack can still surface
> things listed as confidential. Real isolation is architectural — see the
> tenancy design in the product repo. Never mistake this layer for the thing
> keeping one client's data away from another's.

## Three parts, all required

Setting only one of these does not work — the built-in identity wins.

| # | What | Where |
|---|---|---|
| 1 | Identity + guardrails | `SOUL.md` → deployed to `~/.hermes/SOUL.md` |
| 2 | Suppress the engine pointer | `agent.hermes_help_guidance: false` in `~/.hermes/config.yaml` |
| 3 | Recency reinforcement | `agent.system_prompt` in `~/.hermes/config.yaml` |

### 1. `SOUL.md` — the identity slot

`agent/system_prompt.py:387-393` appends `SOUL.md` **instead of**
`DEFAULT_AGENT_IDENTITY` when the file exists. That constant is the
`"You are Hermes Agent, an intelligent AI assistant created by Nous Research."`
line at `agent/prompt_builder.py:151`. With `SOUL.md` present it never enters
the prompt.

This is the load-bearing piece: it occupies the **first** position in the system
prompt, which is where an identity actually sticks.

### 2. The engine pointer

`HERMES_AGENT_HELP_GUIDANCE` (`agent/prompt_builder.py:161`) tells the model it
runs on Hermes Agent by Nous Research and points at those docs. It was appended
unconditionally, which defeats `SOUL.md` on its own — the model reads a Foxwel
identity, then a sentence naming the engine.

We added a gate mirroring the sibling toggles (`task_completion_guidance`,
`parallel_tool_call_guidance`): `agent.hermes_help_guidance`, default `True`.
Set it `false` here.

### 3. The overlay

`agent.system_prompt` is injected at API-call time and appended **last**
(`agent/conversation_loop.py:1568`) — it overlays, it does not replace. It is
deliberately short: the full persona is already at the top and cached, so
repeating it uncached on every request buys little. A few lines at the end buy
recency cheaply.

## Gotcha: the injection scanner will block this file

`SOUL.md` is a context file, so it is scanned before injection into the prompt.
An early draft quoted attack strings verbatim to tell the model what to refuse —
and tripped the `prompt_injection` rule at `tools/threat_patterns.py:65`. The
file loaded as a 95-character `[BLOCKED: ...]` stub and the identity silently
fell back to the default. Nothing errored.

**Describe attack categories; never quote the literal phrases.** After editing,
verify it actually loaded:

```bash
python -c "
import sys; sys.path.insert(0, '.')
from agent.prompt_builder import load_soul_md
s = load_soul_md(200000) or ''
print('BLOCKED' in s, len(s))"
```

Expect `False` and a realistic length. `True`, or ~95 chars, means it was
rejected and the branding is not live.

## Deploying

Not auto-synced — this repo is the source of truth:

```bash
cp branding/SOUL.md ~/.hermes/SOUL.md
~/.hermes/venvs/hermes-dev/bin/hermes gateway restart
```

## Not covered

The CLI startup banner (`skin.branding.agent_name`) still shows the engine name.
Cosmetic and CLI-only; it does not affect Telegram or any messaging surface.

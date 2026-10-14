# MiniMax thinking contract — measured

All results from the live Anthropic-compatible endpoint (`https://api.minimaxi.com/anthropic`),
2026-10-14, fixture-free text/tool probes. Scripts in `/tmp/mm31/probe-thinking.py` and
`/tmp/mm31/probe-interleaved3.py`.

---

## 1. Interleaved thinking is advisory, not enforced

Replaying one assistant turn four ways, all against `MiniMax-M3.1-Flash-Preview`:

| replayed assistant content | result |
|---|---|
| thinking block + **real** signature | **200** |
| thinking block **stripped** (the [hermes-agent #85251](https://github.com/NousResearch/hermes-agent/issues/85251) failure mode) | **200** |
| thinking block + **bogus** signature | **200** |
| thinking block, signature **omitted** | **200** |

Nothing 400s. A made-up signature is accepted. Stripping the block entirely is accepted.

The cost of not adhering is therefore **quality, not failure**. In the stripped case the model
still answers correctly but visibly loses its own chain — the reply begins *"Using arrival times
only… However, this does not show which…"*, i.e. it can no longer see its own prior reasoning.

This is the finding that most changes the build: a harness that drops thinking blocks is not
broken, it is quietly worse.

## 2. `thinking` behaves differently on M3 and M3.1

| model | `thinking` omitted | `{"type":"adaptive"}` | `{"type":"enabled", budget_tokens:N}` | `{"type":"disabled"}` |
|---|---|---|---|---|
| `MiniMax-M3` | **no thinking blocks** | thinking + signature | thinking + signature | stays off |
| `MiniMax-M3.1-Flash-Preview` | thinking on | thinking + signature | — | **400** (2013) |

Two undocumented findings:

- **`{"type":"enabled","budget_tokens":N}` works on M3**, even though MiniMax's docs list only
  `adaptive` and `disabled`. That is the form Anthropic's own driver sends, so pi's default
  path is correct for M3 without any adapter work.
- **`output_config.effort` does not enable thinking on M3.** Sent alone it returns no thinking
  blocks; sent with `adaptive` it does. On M3.1 it tunes depth. Effort is a *tuner*, not a
  *switch*.

## 3. Thinking emission is content-dependent, not effort-dependent

`effort: max` did not force a thinking block on a parallel three-city tool-call turn, and
`effort: low` did produce one on an arithmetic question. A forced
`{"type":"enabled","budget_tokens":8192}` still produced no thinking block on a tool turn.

Consequence for testing: any probe that asserts "thinking must appear" will flake on the same
model and settings. Tests should assert on the **replayed** request, not on emission.

---

## What this means per host

### Pi — M3 needs nothing

`models.dev` publishes `MiniMax-M3` with `reasoning_options: [{"type":"toggle"}]`, which pi's
generator turns into a toggle model (`off` + `minimal`). `off` omits `thinking` → M3 thinks off.
`minimal` sends `{"type":"enabled", budget_tokens:N}` → measured working. Both correct already.

pi's driver already implements the documented contract for every model:

- `INTERLEAVED_THINKING_BETA` (`anthropic-messages.ts:191`), requested by default for
  non-adaptive models (`:1144-1147`)
- thinking blocks round-tripped into assistant messages with their signature, with a guard for
  a missing or empty signature (`:1454-1481`)
- `compat.forceAdaptiveThinking` emits `thinking:{"type":"adaptive"}` **plus**
  `output_config.effort`, and suppresses the beta header (`:1269-1287`)

### Pi — M3.1 is the whole build

Three things, all metadata:

1. a catalog entry at all (absent from pi's generated catalog and from models.dev's
   `minimax-cn`; filed as pi issue [#10916](https://github.com/earendil-works/pi/issues/10916))
2. `compat.forceAdaptiveThinking: true`, so pi emits `adaptive` + `output_config.effort`
3. thinking levels `low | medium | high | xhigh | max`, and **no `off`** — measured 400

### What `pi-minimax-m3-caching-fix` does not cover

It is `MiniMax-M3` **on the OpenAI-compatible endpoint**, and it works on the **output** side —
stripping duplicated and inline `<think>…</think>` from the stream while preserving passive
caching. It provides none of: a catalog entry, the Anthropic route, `adaptive` thinking,
`output_config.effort`, or effort levels. It is orthogonal, not a partial version of this work.

### DSH — likely already done

DSH calls pi's `streamSimple` (`llm-pi-ai/src/adapter.ts:402`), so it inherits pi's thinking
round-trip, and `replay.ts:99,204` preserves `thinkingSignature` across turns. The interleaved
contract is therefore already satisfied.

Two metadata traps remain, both in `llm-pi-ai/src/adapter.ts`:

- **`:131` and `:188` — `off` is translated to *omitting* `reasoning`.** pi then omits
  `thinking`, and MiniMax's server default for M3.1 is thinking **on**. So if `off` is
  selectable, selecting it silently does nothing.
- **`:177` — an effort outside `getSupportedThinkingLevels(model)` throws**
  `does not support reasoning effort "…"`. The model must declare exactly the five levels,
  or the selector offers choices that error.

The profile override already in `~/.dsh/profiles/{docdrift,web}/cordis.patch.yml` declares
`reasoningEfforts: low|medium|high|xhigh|max` with no `off`, which matches the measurement.
What is unverified is whether that override can carry `compat.forceAdaptiveThinking`, and
whether thinking reaches the Web GUI without the known inline `<think>` duplication.

---

## 4. Prompt caching on each endpoint (2026-10-14)

`pi-minimax-m3-caching-fix` exists because its author measured that M3's
`/anthropic/v1/messages` "ignores `cache_control` markers, so every turn was billed at the
full input price ($0.60/Mtok) instead of the cache-read price ($0.12/Mtok)", and that M3
supports passive caching on `/v1/chat/completions`. Measured on `api.minimaxi.com`, 3 turns:

| turn | Anthropic `/anthropic/v1/messages` | | | OpenAI `/v1/chat/completions` | |
|---|---|---|---|---|---|
| | input | cache_read | **cache_create** | prompt | cached |
|---|---|---|---|---|---|
| 1 | 2843 | 128 | **0** | 2978 | 163 |
| 2 | 15 | 2971 | **0** | 2993 | 2977 |
| 3 | 1 | 2985 | **0** | 3008 | 2992 |

**The Anthropic route caches.** Turn 2 is a 99% cache read. The extension's premise — that
M3 bills full input price every turn on `/anthropic` — does not reproduce.

But `cache_creation_input_tokens` is **0 on every turn**, which is the tell: the marker never
created the cache. The cache is created server-side, by content, without being asked. This
matches hermes-agent PR [#84404](https://github.com/NousResearch/hermes-agent/pull/84404)
(merged 2026-08-12) exactly:

> M3 ships server-side automatic prefix caching (content-keyed, no marker needed); explicit
> `cache_control` is documented for M2.7/M2.5/M2.1/M2 only, so markers on M3 were dead
> weight: never observable (`cache_creation_input_tokens` always 0), never billable.

So there was never a caching bug on M3's Anthropic route. The marker was inert; the caching
was automatic either way. PR #17333 (closed) had enabled markers for M2.x, which MiniMax does
document.

## 5. Inline `<think>` duplication is an OpenAI-wire artifact

Raw SSE scan for literal `<think>` / `</think>`, same prompt, same model:

| route | raw stream | inline `<think>` |
|---|---|---|
| `/anthropic/v1/messages` | 8,055 B | **0** |
| `/v1/chat/completions` | 47,732 B | **2** |

The Anthropic wire has a first-class `thinking` block, so MiniMax uses it. The OpenAI wire has
no such block type, so MiniMax inlines the reasoning into `content`. **The duplication the
extension spends ~300 lines removing (a `ThinkScanner`, merged-block dedup, deferred
`text_start`) is caused by the endpoint it routes to in order to fix a caching problem that
was not occurring.**

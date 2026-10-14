# Plan — MiniMax-M3.1-Flash-Preview in Pi

Make `MiniMax-M3.1-Flash-Preview` usable in [Pi](https://github.com/earendil-works/pi) **without**
the video capability. Video is scoped out and tracked separately in
[`pi-video-llm`](../pi-video-llm).

Last updated 2026-10-14. Status: ⬜ not started · 🟡 in progress · ✅ done

---

## 0. The short version

**The gap is one model entry, and it needs no code.** A `models.json` block is enough, and it has
been verified working against the live API. Total cost: **~30 minutes.**

Everything below is the evidence for that claim, because the plan is short and the
justification for it is not.

---

## 1. What is missing, and why it is not a code problem

`MiniMax-M3.1-Flash-Preview` is absent from Pi's generated catalog and from `models.dev`'s
`minimax-cn` entry — it exists in the source data only under a different provider key
(`minimax-cn-coding-plan`). So it cannot be selected.

The obvious workaround does **not** work: `modelOverrides` changes metadata for an *existing*
model, and unknown IDs are silently ignored. A `modelOverrides` entry naming M3.1 is discarded
without an error, so a user following the documented path gets nothing and no message.

What does work is adding a **provider** to `models.json`, which the docs describe as the route
for *"Add models behind a supported API"*. That is the whole fix.

---

## 2. Measured, not inferred

All results from the live Anthropic-compatible endpoint on 2026-10-14. Full transcripts in
[`docs/measurements.md`](docs/measurements.md).

### 2.1 The thinking contract, which is the only fiddly part

| model | `thinking` omitted | `{"type":"adaptive"}` | `{"type":"enabled", budget_tokens:N}` | `{"type":"disabled"}` |
|---|---|---|---|---|
| `MiniMax-M3` | **no thinking blocks** | thinking + signature | thinking + signature | stays off |
| `MiniMax-M3.1-Flash-Preview` | thinking on | thinking + signature | — | **400** (2013) |

- `output_config.effort` accepts `low \| medium \| high \| xhigh \| max`, defaults to `max`.
  `none` returns 400. It **tunes depth; it does not enable thinking** on M3.
- Thinking is **forced on** for M3.1, so there is no `off` and no `minimal`.
- Interleaved thinking is **advisory**: replaying an assistant turn with a real signature, a
  bogus signature, no signature, or with the thinking block stripped **all return 200**. Pi
  already round-trips signatures, so this is satisfied without work.

### 2.2 Pi already has every mechanism needed

- `model.compat.forceAdaptiveThinking === true` makes the Anthropic driver emit
  `thinking: {type:"adaptive"}` **and** `output_config: {effort}`, instead of a
  `budget_tokens` block (`anthropic-messages.ts:1283-1289`).
- `getSupportedThinkingLevels` filters levels by `thinkingLevelMap`: a **`null` value
  suppresses a level**, and `xhigh`/`max` appear **only** when mapped explicitly
  (`models.ts:1230-1239`). `clampThinkingLevel` then falls back to a supported level rather
  than sending an unsupported one.

So the entire declaration is metadata, and the exact spelling is:

```json
"thinkingLevelMap": { "off": null, "minimal": null,
                      "low": "low", "medium": "medium", "high": "high",
                      "xhigh": "xhigh", "max": "max" }
```

### 2.3 Verified end to end, on the real agent

Applied to `~/.pi/agent/models.json` on 2026-10-14 and confirmed in a live interactive `pi`.

```
pi --list-models
  minimax-cn-m31  MiniMax-M3.1-Flash-Preview  1M  512K  yes  yes     ← loaded, no schema error
  volcengine      deepseek-v4-flash-ga-260731   1.0M 393.2K yes  no    ← existing provider intact
```

Every level answers, on the real agent, with empty stderr:

```
off 391   minimal 391   low 391   medium 391   high 391   xhigh 391   max 391
```

`off` and `minimal` returning cleanly is the load-bearing result: `{"type":"disabled"}`
**400s**, so a correct answer from `:off` proves the level was clamped away rather than sent.

A tool round-trip, from the persisted session rather than the transcript — this is where the
interleaved-thinking contract actually gets exercised:

```
toolCall: read call_01a138c4a45b70a
thinking block: 154 chars | signature: 32247fc3e3e48fc0d9a13acb54f2...
```

Signature present and non-empty, so `tool_use` → `tool_result` → continuation completed with
the thinking block replayed intact.

### 2.4 Two traps that cost time, both silent

**The environment gates the provider, not the schema.** With `MINIMAX_CN_API_KEY` unset, the
entry still validates and pi still starts — the provider simply does not appear in `/model`.
Pi only lists providers whose auth resolves, and `apiKey: "$NAME"` resolves from the
environment, not from `auth.json`. An interactive login shell exports it; a bare
non-interactive shell does not.

**`/login` cannot add it, and that is not a failure.** `/login` provisions *OAuth* providers.
An `apiKey` provider declared in `models.json` is not OAuth, so `/login` correctly refuses —
the credential path for it is the environment variable.

### 2.5 Not to be confused with the `modelOverrides` failure

Both failures look identical in `/model`: **nothing appears**. They have opposite causes, and
only one is fixed by deleting the block. Measured in the same file, same session:

| mechanism | M3.1 in `/model` |
|---|---|
| `providers["minimax-cn-m31"].models[]` | **1** |
| `modelOverrides["minimax-cn/MiniMax-M3.1…"]` | **0** |

`modelOverrides` is metadata *for a model that already exists*, so an unknown id is dropped
without an error. A `providers` entry is a real registration — which is why it is listed at
all, and why the earlier `modelOverrides` attempt never was.

---

## 3. Options and cost

| option | what | hours | risk |
|---|---|---|---|
| **A. `models.json` entry** | drop the fragment into `~/.pi/agent/models.json` (and `agent_jane`) | **0.5** | none measured; fully reversible by deleting the block |
| B. Packaged pi extension | `registerProvider` shipping the same entry, reusable via `pi install` | 12 | low, but it re-derives a working config into a published artifact |
| C. Upstream | pi issue [#10916](https://github.com/earendil-works/pi/issues/10916) reopened by a maintainer | 0 | no control over timing; auto-closed once already |

### Option A, itemised

| step | min |
|---|---|
| apply fragment to `~/.pi/agent/models.json`, back it up first | 5 |
| apply to `~/.pi/agent_jane/models.json` | 5 |
| confirm it appears in `/model` in both agents | 5 |
| one live turn per thinking level (`off` `minimal` `low` `medium` `high` `xhigh` `max`) | 10 |

### Option B, itemised

| step | h |
|---|---|
| port the verified entry into `registerProvider`, sibling provider id | 1 |
| live verification across five levels | 2 |
| test suite (Pi mandates its matrix: text, tool calls, usage, abort, overflow, malformed streams) | 4 |
| packaging, README, `pi-package` keyword | 2 |
| DSH parity — `cordis.patch.yml` for each profile | 3 |

Option B is only worth it if you want the entry **portable across machines**. For your own two
agents, Option A is 24× cheaper and does exactly the same thing.

---

## 4. What this does NOT do

- **No video.** `input` stays `["text","image"]`. Video needs a block-type rewrite that cannot
  be expressed in `models.json` at all — `MODEL_INPUT_MODALITIES` is `["text","image"]` and is
  enforced by the schema. See [`pi-video-llm`](../pi-video-llm/PLAN.md).
- **No audio.** MiniMax rejects `type:"audio"` outright (error 2013).
- **No cost change.** M3.1 ships zero cost on M Plan / MiniMax Code, so the entry declares
  zero. If you ever pay per token, that line is the thing to fix.
- **This does not touch DSH.** DSH already has a hand-written M3.1 entry in
  `~/.dsh/profiles/{docdrift,web}/cordis.patch.yml`; the gap this closes is Pi's.

---

## 5. Risks

| risk | likelihood | mitigation |
|---|---|---|
| MiniMax changes the effort vocabulary | low | a rejected effort returns a 400 naming the field; the fix is one line in `thinkingLevelMap` |
| Model is withdrawn (M Plan only) | low | delete the block; nothing else depends on it |
| `thinkingLevelMap` semantics change upstream | very low | the fallback is visible as a 400 on `disabled`, not as silence |
| The user picks a level and gets a silent clamp | **zero by construction** | `clampThinkingLevel` only ever returns a level in `getSupportedThinkingLevels`, which the map defines |

The last row is the one that matters, and it is the behaviour measured in §2.3: because `off`
is mapped to `null` it is never offered, so it can never be sent, so it can never 400.

---

## 6. Success criteria

- [ ] `MiniMax-M3.1-Flash-Preview` selectable in `/model` in both agents
- [ ] all five documented effort levels produce a normal turn
- [ ] no 400 on any level, including the suppressed ones
- [ ] `off` and `minimal` are absent from the picker rather than failing when used
- [ ] a tool call round-trips with the thinking block and its signature preserved

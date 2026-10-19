# pi-minimax-m31

`MiniMax-M3.1-Flash-Preview` in [Pi](https://github.com/earendil-works/pi), without video.

The model is missing from Pi's generated catalog and cannot be added with `modelOverrides`
(unknown IDs are silently ignored). It **can** be added with a `models.json` provider entry —
no code, no extension, no publish.

## Use it

Merge [`models.minimax-cn-m31.json`](models.minimax-cn-m31.json)'s `providers` object into
`~/.pi/agent/models.json`, then restart Pi:

```
/model   ->  minimax-cn-m31 / MiniMax M3.1 Flash Preview
```

Back the file up first. Removing the block reverts everything.

## What is in the entry, and why each field is there

| field | reason |
|---|---|
| `compat.forceAdaptiveThinking: true` | makes the Anthropic driver send `thinking: {type:"adaptive"}` **and** `output_config.effort`. Without it Pi sends `budget_tokens`, which is not MiniMax's M3.1 vocabulary. |
| `thinkingLevelMap.off: null` | thinking is **forced on** for M3.1 — `{"type":"disabled"}` returns 400. Mapping to `null` removes the level from the picker so it can never be sent. |
| `thinkingLevelMap.minimal: null` | `minimal` is not in MiniMax's documented effort set (`low`…`max`). |
| `xhigh` / `max` mapped explicitly | Pi treats these as opt-in: they appear only when the map names them. |
| `cost` all zero | M3.1 is available only through M Plan / MiniMax Code. |
| provider id `minimax-cn-m31` | a sibling, not a shadow of `minimax-cn` — the built-in `minimax-cn/MiniMax-M3` keeps working. |

## Verified

Against the live endpoint, 2026-10-14, in an isolated agent dir:

```
pi --list-models   ->  minimax-cn-m31  MiniMax-M3.1-Flash-Preview  1M  512K  yes  yes
pi -p --model 'minimax-cn-m31/MiniMax-M3.1-Flash-Preview:high' "What is 23*17?"   ->  391
:off  ok    :minimal  ok    :low  ok    :max  Ok.
```

`off` and `minimal` answering cleanly is the point: `disabled` **400s**, so a clean reply
proves they were clamped away rather than transmitted.

## Not in scope

Video is a different problem and a different repo — [`harness-video-llm`](../harness-video-llm).
`MODEL_INPUT_MODALITIES` is `["text","image"]` and enforced by the schema, so no `models.json`
entry can express video. Audio is rejected by MiniMax outright (error 2013).

See [PLAN.md](PLAN.md) for the measurements and the cost comparison against packaging this as
a pi extension.

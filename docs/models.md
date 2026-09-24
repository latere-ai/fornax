# Models

The manifests checked in under [`models/`](../models), what each serving
endpoint answers, and how to add a model. Nothing in the table is a
property of the tool: a model is a manifest, and a deployment's registry
is whatever set of manifests it checks in.

## The checked-in manifests

This is the set Latere is bringing up. **Deploy** is how each one is run
today, not a restriction on it: deploy mode and hardware are independent
fields of the manifest, so a GB10 box can join a cluster and an H200
host can run a model bare-metal. Switching a model's mode is a one-line
manifest edit plus the deploy artifact that mode uses.

| Manifest | Model | Class | Weights | Hardware | Deploy | Served |
|---|---|---|---|---|---|---|
| `minimax-m3` | MiniMax M3 | MoE | MXFP8 | 8x H200 | k8s | not yet |
| `glm-5.2` | GLM-5.2 | MoE | FP8 | 8x H200 | k8s | not yet |
| `kimi-k2.7-code` | Kimi K2.7 Code | MoE | INT4 (QAT) | 8x H200 | k8s | not yet |
| `deepseek-v4-pro` | DeepSeek V4 Pro | MoE | FP4/FP8 | 8x B200 | k8s | not yet |
| `deepseek-v4-flash-0731` | DeepSeek V4 Flash 0731 | MoE | FP4/FP8, DSpark speculative decoding | 8x B200 | k8s | not yet |
| `kimi-k3` | Kimi K3 | MoE, multimodal | MXFP4/MXFP8 | 8x B300 | k8s | not yet |
| `qwen3.8-27b` | Qwen3.8-27B | dense, multimodal | BF16 | 1x GB10 | bare-metal | yes |
| `qwen3.8-27b-fast` | Qwen3.8-27B | dense, multimodal | NVFP4 with a draft head | 1x GB10 | bare-metal | not yet |

"Not yet" means the manifest and its deploy artifact are checked in and
validated, and no endpoint has served from them. Exact revisions,
licenses, and engine arguments are in each manifest.

The two Qwen manifests are the same model on purpose. The BF16 one is
undamaged weights on one GPU and the quality reference; the fast one
trades measured quality for throughput. The precision is in the name and
neither is an alias of the other.

## What each endpoint answers

Every model, in either deploy mode, serves the same paths on one port
(8000 by default). The caller dialects do not vary per model; what
varies is which one the engine speaks natively.

| Path | Dialect | Notes |
|---|---|---|
| `/v1/chat/completions` | OpenAI Chat Completions | native for both SGLang and vLLM |
| `/v1/messages` | Anthropic Messages | the path an unmodified Anthropic SDK requests |
| `/v1/responses` | OpenAI Responses | |
| `/livez`, `/readyz`, `/version` | | `/readyz` answers 503 until the weights are verified and the engine is healthy, and its body names the failing check. `/healthz` and `/ready` are older spellings of `/livez` and `/readyz`, kept for now |
| `/metrics` | | the engine's Prometheus output plus `fornax_*` series |

Any other path is passed through to the engine unchanged, so what the
engine itself serves, such as `/v1/models`, is reachable too.

A caller dialect that matches the engine's own is proxied untouched. The
others translate through
[`latere.ai/x/pkg/llmdialect`](https://github.com/latere-ai/pkg), which
reports every request field the translation could not carry. That report
is returned in the `X-Fornax-Compat-Loss` header and counted in
`fornax_dialect_loss_total`, so a lossy pairing is visible rather than
silent. The engine's own dialect is declared per manifest in
`engine_dialect` and defaults to `openai-chat`.

A model started with a draft head reports it on every response in
`X-Fornax-Speculator`.

No endpoint checks a credential. See
[Security](deploy.md#security) before exposing one.

## Adding a model

1. Write `models/<name>.yaml`: the pinned `hf_repo` and `revision`,
   `format`, `license`, `runtime`, `load`, `gpu`, `context_max`, the
   engine `args`, and `deploy` when it is not `k8s`. Every field is in
   the [manifest reference](deploy.md#model-manifest-modelsnameyaml).
2. Freeze the weights: into a bucket through the
   [mirror Job](deploy.md#2-freeze-the-weights-into-a-bucket), or with
   `fornax pull` and `fornax freeze` for a host that serves from its own
   disk.
3. Add the deploy artifact the mode owns: a LeaderWorkerSet at
   `deploy/<name>/lws.yaml` for `k8s`, or the unit
   `fornax install --print` renders, at `deploy/<name>/<name>.service`,
   for `bare-metal`.
4. Run `fornax validate models/`. It checks the manifest and that the
   artifact matches it. CI runs the same check.

Whether the model belongs on that GPU at all is a separate question:
[sizing a model for a GPU](practice.md) is how to answer it before
spending the download.

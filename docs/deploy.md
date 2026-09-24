# Deploy guide

How to take a model from Hugging Face to a serving endpoint: build the
images, freeze the weights, deploy on GPU Kubernetes or on a single
bare-metal host, and verify. The configuration reference at the end
lists every field and setting.

The two paths are not equally proven. The bare-metal path serves a model
on a GB10 host today. The Kubernetes path is built, and CI checks
every checked-in manifest against its LeaderWorkerSet, but no model has
served on a GPU node yet. Expect to find the first problems on it
yourself.

## Prerequisites

For Kubernetes:

| What | Why | Notes |
|---|---|---|
| An S3 compatible bucket | where frozen weights live | AWS S3, DigitalOcean Spaces, Cloudflare R2, MinIO, anything `s5cmd` reaches. Turn on versioning, and Object Lock if the provider has it. The checked-in manifests name a bucket called `latere-models`; change `s3_prefix` in `models/*.yaml` to your own. |
| A Secret `mirror-s3` in the namespace `fornax` | credentials for the mirror Job and the node cache | keys `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, plus `S3_ENDPOINT_URL` for anything that is not AWS. |
| GPU nodes and the NVIDIA GPU Operator | run the engines | the checked-in artifacts select node pools by the label `latere.ai/gpu-pool` with the values `h200`, `b200`, or `b300`, and expect NVMe at `/var/cache/fornax`. Change the label in `deploy/*/lws.yaml` to match your nodes. The `b300` pool needs an r580 or newer driver, because the Kimi K3 image is CUDA 13 only. |
| [LeaderWorkerSet](https://github.com/kubernetes-sigs/lws) | the pod-group primitive serving runs as | `kubectl apply --server-side -f https://github.com/kubernetes-sigs/lws/releases/latest/download/manifests.yaml` |
| A container registry you can push to | the images | `ghcr.io/latere-ai` by default; any OCI registry works through `REGISTRY=` |

For a bare-metal host: Linux with systemd, an NVIDIA GPU with its
driver, and the engine installed as described in
[section 4](#4-bare-metal-hosts). No bucket and no cluster.

## 1. Build and push images

No images are published yet, so build them. Four images, versioned
together:

```sh
make push-images VERSION=v0.1.0 REGISTRY=registry.example.com/fornax
```

builds `linux/amd64` images and pushes them. The image names are fixed;
the registry prefix is yours.

| Image | What it runs |
|---|---|
| `fornax-runtime-sglang` | SGLang `v0.5.16-cu129` and the `fornax serve` entrypoint |
| `fornax-runtime-sglang-k3` | a Kimi K3 capable SGLang build (CUDA 13, r580 or newer driver), kept separate so that driver floor does not reach the other pools |
| `fornax-runtime-vllm` | vLLM `v0.28.0` and the `fornax serve` entrypoint, including the `load: s3-stream` path |
| `fornax-mirror` | the `fornax` binary with `hf` and `s5cmd`, for the weight-freeze Job |

Engine versions are pinned in `Dockerfile.sglang` and `Dockerfile.vllm`.
Bump them deliberately and never to `latest`.

Then point the deploy artifacts at what you pushed: the `image:` lines in
`deploy/*/lws.yaml` and `deploy/mirror/job.yaml` name
`ghcr.io/latere-ai/...:v0.1.0`, which does not exist. `fornax validate`
checks the image name against the manifest's runtime and ignores the
registry, so a custom registry passes the check unchanged. If your
registry is private, add `imagePullSecrets` to the pod specs.

## 2. Freeze the weights into a bucket

Once per model revision. Run it in the cluster, where the bandwidth and
the disk are:

```sh
# Edit deploy/mirror/job.yaml: set metadata.name, MODEL_REPO, MODEL_SHA,
# --bucket, and the scratch volume size (at least the model's size on disk).
kubectl -n fornax apply -f deploy/mirror/job.yaml
kubectl -n fornax logs -f job/mirror-<name>
```

The Job resolves the revision, lists the repository's files, downloads
them, checks each against its published size and, for large files,
the SHA-256 the Hub publishes, hashes the rest itself, uploads through
`s5cmd`, and writes `_manifest.json` last. Its presence
marks the mirror complete. Running the Job again is safe: files already
verified are not fetched again. Check a mirror at any time:

```sh
fornax verify s3://<your-bucket>/<org>/<repo>/<sha>/
fornax list --bucket s3://<your-bucket>
```

Two rules decide what can be frozen:

- **Safetensors only.** A repository whose files include anything but
  `.safetensors` weights and known non-weight companions (configuration,
  tokenizer files, documentation, source) is refused before anything is
  downloaded, and the error names the file.
- **Public repositories only, for now.** A gated repository works: its
  metadata is public, and `hf download` reads `HF_TOKEN` for the files,
  so put the token in the `mirror-s3` Secret. A private repository does
  not: the revision and file-list requests `fornax` makes before the
  download send no token, so it fails at the first request.

Then pin the model in `models/<name>.yaml` (see the
[manifest reference](#model-manifest-modelsnameyaml)) and run
`make validate`. CI refuses a manifest without a consistent
`deploy/<name>/lws.yaml`.

## 3. Deploy and serve

```sh
kubectl create namespace fornax --dry-run=client -o yaml | kubectl apply -f -

# The runtime reads the manifest from a ConfigMap:
kubectl -n fornax create configmap kimi-k2-7-code-manifest \
  --from-file=model.yaml=models/kimi-k2.7-code.yaml

kubectl -n fornax apply -f deploy/kimi-k2.7-code/lws.yaml
```

Watch the start. The pod stages the weights from the bucket onto the
node's NVMe, verifies them, then launches the engine:

```sh
kubectl -n fornax get pods -w
kubectl -n fornax logs -f <pod>   # "weights: fetching ..." then "launching sglang"
```

`/readyz` answers 503 while the model loads and 200 once the engine is
healthy. The readiness probe allows a long cold start, and a restart on
the same node skips the download, because the cache is keyed by
repository and revision. Then try each surface:

```sh
kubectl -n fornax port-forward svc/kimi-k2-7-code 8000 &

# OpenAI Chat: proxied to the engine untouched
curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"kimi-k2.7-code","max_tokens":64,"messages":[{"role":"user","content":"hello"}]}'

# Anthropic Messages: translated
curl -s localhost:8000/v1/messages -H 'Content-Type: application/json' \
  -d '{"model":"kimi-k2.7-code","max_tokens":64,"messages":[{"role":"user","content":"hello"}]}'

# OpenAI Responses: translated
curl -s localhost:8000/v1/responses -H 'Content-Type: application/json' \
  -d '{"model":"kimi-k2.7-code","input":"hello"}'

# Metrics: the engine's, plus fornax_weights_load_seconds
curl -s localhost:8000/metrics | grep fornax

# A baseline benchmark, the figures a gateway's cost configuration needs
fornax bench --url http://localhost:8000 --model kimi-k2.7-code \
  --concurrency 8 --requests 32 --out report.json
```

Finally register the in-cluster endpoint,
`http://<name>.fornax.svc:8000/v1`, as a provider in your model gateway
([Lux](https://github.com/latere-ai/lux) is the one Fornax is built
beside). Do not expose the Service directly; see [Security](#security).

**Licenses.** Check `license` and `license_note` in each manifest before
you expose a model. Among the checked-in set: MiniMax M3 requires a
one-time commercial notice before commercial use, Kimi K3's license
restricts offering the model as a service, so decide whether your use
counts as internal before exposing it, and Kimi K2.7 carries a
modified-MIT attribution clause. The DeepSeek V4 checkpoints and GLM-5.2
are MIT, and Qwen3.8-27B is Apache-2.0.

## 4. Bare-metal hosts

A single GPU box with no cluster runs the same manifests through an
installed binary and a systemd unit. A model opts in with
`deploy: bare-metal`; `deploy: k8s` is the default, so nothing else
changes.

```sh
fornax install --manifest models/qwen3.8-27b.yaml --cache-root ~/.models --user <user>
sudo systemctl enable --now qwen3.8-27b.service
fornax ps
```

`install` writes `<name>.service` to `/etc/systemd/system` and copies the
manifest to `/etc/fornax/<name>.yaml`, then runs `systemctl
daemon-reload`. It changes nothing when the manifest has not changed.

| Flag | Default | Effect |
|---|---|---|
| `--manifest` | required | the model to install |
| `--cache-root` | none, so `serve` uses `/cache` | the weights root passed to `serve` |
| `--user` | none, so root | the user the service runs as |
| `--speculator` | the manifest's `default_speculator` | pin a draft head into the unit; see below |
| `--bin` | `/usr/local/bin/fornax` | the installed binary the unit runs |
| `--config-dir` | `/etc/fornax` | where the manifest is placed |
| `--unit-dir` | `/etc/systemd/system` | where the unit is written |
| `--print` | off | write nothing and print the unit that would be installed |
| `--no-reload` | off | write the files and leave systemd alone, for staging a unit meant for another machine |

The unit restarts the model on failure and allows it 30 minutes to start,
because a large model takes minutes to load and systemd's default 90
seconds kills it mid-load and then keeps killing it.

Weights come from local disk (`load: local`). `fornax pull` and
`fornax freeze` place them under `<cache-root>/<hf_repo>/<revision>`,
and `serve` verifies them there against `_manifest.json` without
copying. No bucket is involved.

### Install the engine yourself

Fornax does not install engines on a bare-metal host. Install the engine
into a virtualenv. The GB10 manifests checked in here are served with
these versions, which differ from the container pins above:

```sh
uv venv ~/.venvs/fornax-vllm
uv pip install --python ~/.venvs/fornax-vllm/bin/python vllm==0.28.0

uv venv ~/.venvs/fornax-sglang
uv pip install --python ~/.venvs/fornax-sglang/bin/python sglang==0.5.18
```

**Give each engine its own virtualenv.** Installing SGLang beside vLLM
downgrades `transformers` and `xgrammar`, and vLLM then stops working.
The dependency sets are incompatible, not merely untested. A box that can
serve either model needs both environments, even though it serves one
model at a time.

`serve` launches `vllm serve` for a `vllm` manifest and
`python3 -m sglang.launch_server` for an `sglang` one, found on the
service's `PATH`. The generated unit sets no `PATH`, so give each unit
its engine's environment with a drop-in, which survives a later
`fornax install`:

```sh
sudo systemctl edit qwen3.8-27b.service
# [Service]
# Environment=PATH=/home/<user>/.venvs/fornax-vllm/bin:/usr/local/bin:/usr/bin:/bin
```

### Make the host recoverable before you serve on it

A single GPU box with unified memory can be taken down by a
configuration mistake, not only a hardware fault. The CPU and GPU draw
from one pool, so an engine that claims too much of it leaves the kernel
unable to reclaim memory, and the machine stops answering SSH instead of
killing the engine and staying up. If the box is remote, that costs you
the machine until someone walks to it.

Two host settings turn that from "down until a visit" into "back in a
minute". Apply them before the first serve:

```sh
# 1. Hardware watchdog: reset the box if the kernel stops responding.
ls /dev/watchdog*                       # confirm the device exists first
sudo sed -i 's/^#\?RuntimeWatchdogSec=.*/RuntimeWatchdogSec=60s/' /etc/systemd/system.conf
sudo sed -i 's/^#\?RebootWatchdogSec=.*/RebootWatchdogSec=5min/'  /etc/systemd/system.conf
sudo systemctl daemon-reexec

# 2. Reboot on panic and on out-of-memory, rather than hanging.
printf 'kernel.panic=10\nkernel.panic_on_oops=1\nvm.panic_on_oom=0\n' \
  | sudo tee /etc/sysctl.d/99-fornax-recover.conf
sudo sysctl --system
```

`RuntimeWatchdogSec=60s` has systemd service the hardware watchdog every
30 seconds; if the kernel stalls, the watchdog fires and the box reboots
on its own. That is the setting that matters when you cannot reach the
power button.

Keep the journal across reboots, or you lose the evidence for why one
happened:

```sh
sudo mkdir -p /var/log/journal && sudo systemd-journal-flush
journalctl -k -b -1 | tail -60      # the previous boot, after a crash
```

Keep diagnostics out of `/tmp` on such a host: it is cleared on boot,
which is exactly when you need them.

Fornax also guards the common case itself. The manifest check refuses a
GB10 manifest that does not state the engine's memory fraction and
context length, or that sets the fraction above 0.80. `fornax serve`
then refuses to start when the memory fraction, plus the checkpoint it
just read, plus an 8 GB floor for the host, would exceed the machine's
memory. The settings above cover what neither check can predict.

### Choosing a draft head

A model that declares `speculators` starts with its
`default_speculator`, and you can pick another when it starts, because
which one is fastest depends on the workload; see
[the practice notes](practice.md#choosing-a-draft-head).

```sh
fornax serve   --manifest models/qwen3.8-27b-fast.yaml --speculator dflash2
fornax install --manifest models/qwen3.8-27b-fast.yaml --speculator dflash2
```

`install --speculator` pins the choice into the unit, so a restart
serves what you installed rather than the manifest's default. An unknown
name fails before anything is written. `--speculator none` serves the
model with no draft head.

`fornax ps` reports which head each model is running, and every response
carries `X-Fornax-Speculator`. Quote it with any throughput figure.

One GPU serves one model at a time, so changing models or draft heads
means stopping the running unit first.

To use a served model from Claude Code, Codex, or opencode on the same
host, see [coding agents](coding-agents.md).

## Security

`fornax serve` checks no credential. Anyone who can reach the port can
use the model, and read `/metrics`. The engine behind it listens on all
interfaces as well, on port 30000 by default, with no authentication of
its own.

- On Kubernetes, keep the Service cluster-internal and let a model
  gateway be the only way in. The gateway holds the credentials, the
  quotas, and the audit trail.
- On a bare-metal host, keep both ports off public networks: bind the
  host to a private network or firewall 8000 and 30000 to the callers you
  mean.

`fornax endpoint` and `fornax run` emit a placeholder token only because
some clients refuse to start without one. It protects nothing.

## Local rehearsal

The whole pipeline runs on a laptop at no cloud cost, with the same
manifests and tools: MinIO for the bucket, a 0.6B model, and `mlx_lm` as
the engine.

```sh
make e2e-local
```

Use it to check a change to the runtime or the mirror before touching
real hardware. It needs Docker or Podman, `uv`, Apple silicon for mlx,
and a one-time download of about 1.5 GB.

## Configuration reference

### Model manifest (`models/<name>.yaml`)

The manifest is the source of truth for a deploy. `fornax validate`
reports every problem in one message.

| Field | Values | Effect |
|---|---|---|
| `name` | `[a-z0-9.-]+`, required | the model id callers use. Kubernetes resources use it with `.` replaced by `-` |
| `hf_repo`, `revision` | `<org>/<name>` and a 40-hex commit SHA, required | the pinned identity. `revision` must also appear in `s3_prefix` |
| `s3_prefix` | `s3://<bucket>/<hf_repo>/<revision>/` | where the frozen weights live. Required unless `load: local`, which requires it empty |
| `format` | free text (`fp8`, `int4-qat`, ...), required | a record of the checkpoint format |
| `license`, `license_note` | free text; `license` required | the compliance record. A gate noted here is yours to honor before exposing the model |
| `runtime` | `sglang`, `vllm`, or `custom`, required | which engine image runs. `custom` requires `image` and serves any container that answers the health contract |
| `image` | an image reference | the container for `runtime: custom`, and allowed only there |
| `engine_dialect` | `openai-chat` (default), `anthropic-messages`, or `openai-responses` | the wire dialect the engine itself speaks. All three caller surfaces are served whatever it is; the matching one is proxied untouched, the others are translated, and what a translation dropped is reported in `X-Fornax-Compat-Loss` and `fornax_dialect_loss_total` |
| `deploy` | `k8s` (default) or `bare-metal` | which deploy artifact the model owns: `deploy/<name>/lws.yaml` or `deploy/<name>/<name>.service`, never both. `bare-metal` does not allow `image` |
| `load` | `nvme-cache`, `s3-stream`, or `local`, required | stage the weights on node NVMe; stream them from the bucket (vLLM only); or verify them in place on the host's disk with no bucket |
| `gpu` | `{type, count, nodes}`, required | the resource shape; must match the LeaderWorkerSet. `type: gb10` requires `count: 1` and `nodes: 1` |
| `context_max` | a positive integer, required | the context the model is configured for; pair it with the KV cache arguments it needs |
| `args` | a list, passed verbatim; required for `sglang` and `vllm` | the engine flags for this model: parallelism (`--tp-size`), parsers (`--tool-call-parser`), quantization, KV dtype. Some are enforced per model: MiniMax M3 needs `--block-size=128`, and Kimi K3 and DeepSeek V4 Flash 0731 need `--trust-remote-code`. With `--speculative-algorithm DSPARK`, a separate `--speculative-draft-model-path`, `--pp-size` above 1, and DP attention are refused. On `gpu.type: gb10` the memory fraction (`--gpu-memory-utilization` or `--mem-fraction-static`) and the context bound (`--max-model-len` or `--context-length`) must be stated, and the fraction may not exceed 0.80 |
| `speculators` | a map of name to `{hf_repo, revision, s3_prefix, license, license_note, args}` | the draft heads the model offers; `--speculator <name>` selects one. An entry naming an `hf_repo` is a separately published head: it pins its own revision and states its own license, and is frozen and verified like the main weights. An entry with only `args` selects a head inside the model's checkpoint. The draft path is resolved at launch to `<cache-root>/<hf_repo>/<revision>`, never written here. These `args` come after the model's own, so they win |
| `default_speculator` | a `speculators` key, or `none` | the head that runs when none is named. Required whenever `speculators` is set |
| `system_prompt` | `{mode, text}` | applied by the shim to every request on every surface. `mode` is `default` (only when the caller sends none), `prepend`, or `override` |

The runtime renders the base engine command itself and always adds
`--served-model-name <name>`, so `args` carries only what is specific to
the model.

### `fornax serve`

| Setting | Default | Purpose |
|---|---|---|
| `--manifest` | required | the manifest path. In a container, the mounted ConfigMap at `/etc/fornax/model.yaml` |
| `--port` | `8000` | the port the probes, `/metrics`, and the three caller surfaces answer on |
| `--engine-port` | `30000` | the engine's own port |
| `--cache-root` | `/cache` | the weights root. On Kubernetes, the node NVMe mount, keyed by repository and revision and shared between pods on a node under a file lock |
| `--speculator` | the manifest's `default_speculator` | the draft head to serve with; `none` turns speculation off. Resolved before any weights are touched |
| `FORNAX_ENGINE_CMD` | unset | replaces the engine command, with `{model}` and `{port}` substituted. For a local engine such as mlx |
| `FORNAX_ENGINE_HEALTH_PATH` | `/health` | the engine's health path, for an engine that differs |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | unset | the OTLP collector traces, metrics, and logs go to. Unset, `serve` logs locally and exports nothing |

### Weight commands

| Setting | Default | Purpose |
|---|---|---|
| `FORNAX_HF_BASE` | `https://huggingface.co` | the Hub address `fornax` resolves revisions and lists files at. `hf download` does not read it; point that tool at the same place with its own `HF_ENDPOINT` |
| `HF_TOKEN` | unset | read by `hf download` only, which is enough for a gated repository and not for a private one; see [section 2](#2-freeze-the-weights-into-a-bucket) |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_ENDPOINT_URL` | unset | read by `s5cmd` for an `s3://` store |

A `--bucket` or a store prefix that does not start with `s3://` is a
local directory, which is how a single host or a test keeps a store with
no bucket.

### Telemetry

`fornax serve` exports traces, metrics, and logs over OTLP. Point it at a
collector with the standard environment:

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=https://collector.example:4318
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer%20...   # percent-encoded; %20 is the space
OTEL_TRACES_SAMPLER_ARG=0.2                             # head sampling ratio
```

With the endpoint unset the exporters do nothing and logs stay on
stderr, so a laptop or a bare-metal host needs no collector.

Each request produces a server span for the caller's request and a client
span for the engine call inside it, so a slow completion shows which half
was slow. The probes and `/metrics` are not traced, because the kubelet
and `fornax ps` poll them constantly.

`/metrics` stays a Prometheus text endpoint, because `fornax ps` reads
`fornax_weights_load_seconds` from it. The same three facts also export
as OTLP instruments: `fornax.weights.load.duration`,
`fornax.speculator.info`, and `fornax.dialect.loss`.

### LeaderWorkerSet (`deploy/<name>/lws.yaml`)

| Setting | Where | Notes |
|---|---|---|
| replicas | `spec.replicas` | whole serving groups; capacity planning, not autoscaling |
| group size | `leaderWorkerTemplate.size` | equals `gpu.nodes`. Above 1 is multi-node serving, which needs RoCEv2 and NCCL, and requires a `workerTemplate` with the same image and GPU count as the leader and no probes, since only rank 0 serves HTTP. Checked by `fornax validate` |
| GPU count and pool | `resources.limits."nvidia.com/gpu"`, `nodeSelector` | must match the manifest's `gpu`, checked by `fornax validate` |
| image | the container `image` | `<REGISTRY>/fornax-runtime-<engine>:<VERSION>` from `make push-images`. The registry is free; the name must match the manifest's runtime, checked by `fornax validate` |
| NVMe cache | `volumes.cache.hostPath` | `/var/cache/fornax`. The first pod on a node fills it; later pods for the same revision reuse it |
| `/dev/shm` | `volumes.shm.sizeLimit` | at least 32Gi; vLLM needs it for DeepSeek V4 class models |
| probe budget | `readinessProbe.failureThreshold` | a cold start of a large model takes minutes, so size it for that |

### Mirror Job (`deploy/mirror/job.yaml`)

| Setting | Purpose |
|---|---|
| `MODEL_REPO`, `MODEL_SHA` | the revision to freeze |
| the `mirror-s3` Secret | bucket credentials; `S3_ENDPOINT_URL` for Spaces, R2, or MinIO |
| scratch volume size | at least the model's size on disk: 167 GB to 1.6 TB across the checked-in set, with Kimi K3 alone at 1561 GB |

## Troubleshooting

- **`/readyz` stays at 503.** The body names the check. In the pod or
  unit log, `weights: fetching` is a normal cold start; an engine crash
  shows the engine's stderr; a hash mismatch means the store is damaged,
  so run `fornax verify` on it.
- **404 from `/v1/chat/completions`.** The `model` in the request must be
  the manifest's `name`, which is the name the engine serves under.
- **The mirror Job fails mid-upload.** Run it again. The push skips files
  already verified, and a missing `_manifest.json` means the mirror is
  incomplete.
- **A second pod on the same node downloads again.** The cache is keyed
  by repository and revision under `--cache-root`; check the `hostPath`
  mount and that the revisions match.
- **`fornax pull` fails with `401` or `404` on the first request.** The
  repository is private, or the name or revision is wrong; see
  [section 2](#2-freeze-the-weights-into-a-bucket). A `403` from
  `hf download` on a gated repository means `HF_TOKEN` is missing or has
  not been granted access.
- **`fornax serve` exits at once with `executable file not found`.** The
  engine is not on the service's `PATH`; see
  [installing the engine](#install-the-engine-yourself).

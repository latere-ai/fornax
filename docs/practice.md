# Sizing a model for a GPU

For an operator deciding whether a model belongs on a machine, before
spending the download. Two questions decide it, and they have different
answers:

1. **Does it fit?** Decided by *total* parameters.
2. **How fast will it be?** Decided by *active* parameters.

Conflating them is the usual mistake. A 300B mixture-of-experts model
with 13B active is harder to fit than a 27B dense model and much faster
once it does.

The measured figures below come from serving the checked-in models on a
GB10 host, and the arithmetic applies to any GPU.

## 1. Does it fit?

```
weights + kv_cache + activations  ≤  fraction × pool
```

`weights` is total parameters times bytes per parameter:

| Precision | Bytes/param | 27B | 300B | 1T |
|---|---|---|---|---|
| BF16/FP16 | 2 | 54 GB | 600 GB | 2 TB |
| FP8 | 1 | 27 GB | 300 GB | 1 TB |
| INT8 | 1 | 27 GB | 300 GB | 1 TB |
| 4-bit | 0.5 | 13.5 GB | 150 GB | 500 GB |
| 2-bit | 0.25 | 6.75 GB | 75 GB | 250 GB |

`kv_cache` is per token, times the context you intend to serve:

```
kv_per_token = layers_with_full_attention
             × kv_heads × head_dim × 2 (K and V) × bytes_per_element
```

Read those from the checkpoint's `config.json`, never from the model's
name. Two things routinely surprise:

- **Hybrid-attention models cache on only some layers.** Qwen3.8-27B has
  64 layers, but only 16 use full attention; the other 48 are Gated
  DeltaNet, whose recurrent state has a fixed size and does not grow with
  the sequence. That is a 4x reduction in KV, and it is why a 262K
  context is affordable on one GPU. Predicted 64 KiB per token, measured
  65.4: the formula holds when you read the real config.
- **Compressed-KV designs** (MLA and similar) break the formula in the
  other direction. If the config does not give you the geometry, measure
  rather than guess.

`activations` is a few GB for a single-stream server; 5 GB was the
measured peak for a 27B. It grows with batch size.

## 2. How fast will it be?

Token generation at batch 1 is bound by memory bandwidth. The model
reads its weights once per token:

```
tok/s  ≈  efficiency × bandwidth / bytes_read_per_token
```

- **Dense model:** `bytes_read` is *all* the weights.
- **MoE model:** `bytes_read` is the *active* weights only, because the
  router touches a fraction of the experts per token.
- **efficiency** is 60 to 75% in practice. The rest goes to attention,
  sampling, scheduling, and kernels that are newer and less tuned.

This is why active parameters, not total, set the speed. It is also why
quantization buys throughput proportionally: half the bytes, twice the
tokens.

### Speculative decoding breaks this formula

The formula assumes **one token per forward pass**, the only case where
bytes read per token is a fixed cost. Speculative decoding reads the
weights once and verifies several drafted tokens against that read, so
the honest form is:

```
tok/s  ≈  efficiency × bandwidth / bytes_read_per_token × accepted_per_step
```

The multiplier is not a constant. It is how often the draft model
guesses right, which depends on what is being generated. Published
figures for Qwen3.8-27B on a GB10, with the same model and precision
throughout:

| Content | tok/s |
|---|---|
| Code | 51.5 |
| Long essay | 18.3 |

**A 2.8x spread on identical hardware.** Memory bandwidth is indifferent
to what is being written; draft acceptance is not, because code is far
more predictable than prose. A throughput figure quoted without saying
what was generated is probably the code figure.

Two consequences for planning:

- A bandwidth ceiling is a floor for *speculative* serving, not a cap.
  Qwen3.8-27B measured 3.0 tok/s at BF16 with no speculation, against a
  ceiling of 4.3, and 51.5 with 4-bit weights and a draft head: ten times
  that ceiling, and not a contradiction.
- Quantization and speculation **multiply**. Four times fewer bytes and
  about four times the accepted tokens is the 17x between those two
  figures.

### Choosing a draft head

A model usually has more than one draft head published for it, and there
is no single winner. The three for Qwen3.8-27B, on the same weights and
the same GPU:

| Draft head | Code | Prose | Fetched |
|---|---|---|---|
| DSpark | **51.5** | 18.3 | 2.7 GB |
| DFlash2 | 50.9 | **25.4** | 2.6 GB |
| MTP | 34.5 | 24.1 | nothing; it is in the checkpoint |

DSpark wins on code by 17 tok/s and loses on prose by 7, so the head is
chosen for the workload when the model starts rather than fixed in the
manifest:

```
fornax serve --manifest models/qwen3.8-27b-fast.yaml --speculator dflash2
fornax serve --manifest models/qwen3.8-27b-fast.yaml --speculator none
```

`fornax ps` shows which one is running, and every response carries an
`X-Fornax-Speculator` header. Record it with any throughput figure: the
same endpoint gives a different figure with a different head, so a
measurement that omits it cannot be compared to anything.

Bring a newly quantized model up with `--speculator none` first. That
serves the quantized weights with no draft head, which separates the two
things that changed: what quantization cost in quality, and what
speculation bought in speed. Measured together, they tell you neither.

### Measuring bandwidth, and a trap

Do not measure bandwidth with a reduction. `tensor.sum()` on a GB10
reports 164 GB/s and is limited by the reduction, not by memory, so it
makes the hardware look slower than it is. A copy, or an elementwise
operation over distinct arrays, is the honest measure, counting the
bytes actually moved:

```python
import torch, time
n = 200_000_000
a = torch.ones(n, dtype=torch.bfloat16, device="cuda"); b = torch.empty_like(a)
for _ in range(5): b.copy_(a)
torch.cuda.synchronize()
t0 = time.perf_counter(); b.copy_(a); torch.cuda.synchronize()
print("%.0f GB/s" % (n*2*2 / (time.perf_counter()-t0) / 1e9))  # read + write
```

## 3. Unified memory changes the rules

On a part where the CPU and GPU share one pool (GB10 and similar), four
things differ from a node with discrete HBM:

- **A memory *fraction* is taken from the operating system's memory
  too.** vLLM at `--gpu-memory-utilization 0.30` held 37.6 GB on a
  128 GB box: 0.30 of the *whole pool*, for a model whose weights were
  1.2 GB. The default of 0.90 would leave the host about 13 GB. Cap it.
  Fornax refuses a GB10 manifest above 0.80, and the checked-in ones sit
  below that.
- **The engine fills the fraction it is given**; it does not stay under
  it. Asking for less is how you leave the host room.
- **`nvidia-smi` reports no device memory** on this class: `[N/A]` for
  total, free, and used. Read the host's `/proc/meminfo` instead.
  Anything that sizes a cache from device-memory queries reads zero.
- **CPU offload buys nothing, and the saving it advertises is not
  real.** This is the least obvious of the four.

### Why "offload to host RAM" does not help here

A common trick for a model that does not fit is to keep part of it in
host RAM: an embedding table, cold experts, whatever is touched least.
On a discrete GPU that is a real saving, because host RAM is a second,
separate pool.

On unified memory there is no second pool. Host RAM *is* GPU memory, so
moving a tensor from one to the other moves nothing, and the requirement
is the whole checkpoint either way.

Qwen3.8-Flash-Next is the worked example. Its vLLM recipe says:

> Host RAM: at least 51 GB for N-gram embedding offload

On a discrete GPU that turns a 172.8 GiB FP8 checkpoint into about
122 GiB of GPU residency plus 51 GB of host RAM. On a 128 GB unified box
it stays 172.8 GiB against 128 GB, and does not fit.

**So a unified-memory part is *worse* than a discrete GPU of the same
stated capacity for any model whose plan depends on offload**, which is
the opposite of the intuition that sharing a pool is more flexible. When
a model's sizing guidance names host RAM as a separate budget, add the
two figures together before deciding it fits.

## 4. A GB10 host, measured

One GB10, 128 GB of unified LPDDR5X, an arm64 host, no cluster.

| | Measured |
|---|---|
| Usable memory | about 115 GB (less with a desktop session) |
| Engine budget at the 0.80 cap | about 102 GB |
| Memory bandwidth | **about 230 GB/s** (copy; see the trap above) |
| Inference efficiency | about 70% of that |
| NVMe read | 4.9 GB/s |
| Compute capability | sm_121, running sm_120 kernels by binary compatibility |
| Weight verification | 54 GB in 39 s (about 1.4 GB/s hashing) |
| Engine start | 382 s for a 27B; about 10 minutes to `/readyz` |

**Startup is the operational surprise.** systemd's default start timeout
of 90 seconds kills a large model mid-load and then keeps killing it,
which looks like a crash loop rather than a timeout. The units
`fornax install` writes allow 30 minutes.

## 5. Worked examples

Models evaluated for a GB10, with the reasoning that decided each.
`bytes/token` uses *active* parameters at the serving precision.

| Model | Total | Active | Size to serve | Fits 102 GB? | bytes/token | Expected tok/s |
|---|---|---|---|---|---|---|
| Kimi K3 | 2.8T | 104B | about 1.4 TB at 4-bit | **no**, 14x over | | |
| DeepSeek V4 Pro | 1.6T | 49B | 1.6 TB at FP8 | **no**, 16x over | | |
| Qwen3.8-Flash-Next | 125B (+55B) | 6B | 185 GB at FP8 | **no**, 1.4x over | about 3 GB | |
| DeepSeek V4 Flash | 304B | 13B | 90.9 GB at IQ2_M | tight, yes | about 4 GB | high; see below |
| **Qwen3.8-27B** | 27B dense | 27B | 54 GB at BF16 | **yes**, comfortably | **54 GB** | **3.0 measured** |

### Qwen3.8-Flash-Next: two separate blocks, and only one is hard

Worth separating, because they lead to different decisions.

**The engine version is soft.** The architecture
(`Qwen4ExpForConditionalGeneration`) is supported by vLLM from
**0.29.0**; the 0.28.0 the checked-in images pin rejects it with
`Model architectures [...] are not supported for now`. That message
reads like "impossible" and means "your engine is older than this
model". A version bump fixes it.

**The memory is hard.** The official FP8 checkpoint is 172.8 GiB and its
recipe states a minimum of two GB300s. The BF16 one is 335 GiB. Neither
fits 128 GB, and the recipe's 51 GB of host RAM for N-gram embedding
offload, which would leave about 122 GiB resident on a discrete GPU,
saves nothing here, for the reason in section 3.

Note the `bytes/token` column: **about 3 GB** at FP8 for 6B active. If a
sub-4-bit quantization appears, this is the fastest model in the table
by a wide margin, and the only thing between it and this box is
capacity.

### What the table shows that each decision alone did not

**Qwen3.8-27B reads 54 GB per token. DeepSeek V4 Flash at 2-bit reads
about 4 GB.** That is a 13x difference in the quantity that sets speed,
and it runs opposite to the quality argument.

The case for Qwen over the compressed DeepSeek was that undamaged weights
beat a large model squeezed to fit. That argument stands on quality. It
does not stand on throughput, and the gap is not marginal: it is the
difference between a model you wait on and one you converse with. Weigh
both when choosing.

### Qwen3.8-27B in detail

Predicted before deployment, then measured:

| | Predicted | Measured |
|---|---|---|
| KV per token | 64 KiB | 65.4 KiB |
| Weights resident | 54.0 GB | 56.0 GiB (including non-torch) |
| Activation peak | about 6 GB | 4.9 GiB |
| Engine total | at most 83.2 GB | 71.4 GiB |
| Throughput | not predicted | 3.0 tok/s (about 70% of the 4.3 ceiling) |

The arithmetic was reliable; the throughput was what running it taught.

### The quantization ladder for a 27B dense model

At the same efficiency of about 70%:

| Precision | bytes/token | Expected tok/s |
|---|---|---|
| BF16 | 54.0 GB | 3.0 |
| INT8 | 27.0 GB | about 6 |
| 4-bit | 13.5 GB | about 12 |

INT8 is the interesting middle: half the bytes, for a quality cost far
below 4-bit's, on a model with no quantization-aware training to lean
on.

Add a draft head on top and the 4-bit row becomes about 50 tok/s on
code, which is what the `qwen3.8-27b-fast` manifest is built for. The
two levers are independent and multiply, so treating quantization as the
only one undersells the ceiling by roughly the acceptance rate.

## 6. Operating a model once it is up

### Restarting is not a cheap edit

A manifest change means a restart, and a restart means reloading the
weights. On a GB10 that is **about 10 minutes** for a 27B model: 39 s to
verify 54 GB in place, then about 380 s of engine start before
`/readyz`.

Batch manifest changes rather than changing one flag at a time, and
expect the endpoint to be gone for the duration: there is one GPU, so
there is no rolling restart to hide behind.

### Never match a process by a string your own command contains

Over SSH, the remote command line is visible in the process table. With
Tailscale SSH it appears inside `tailscaled be-child ssh ... --cmd=<your
whole command>`, and plain `ssh host '<cmd>'` exposes it too. So a `-f`
match on any string in your command finds *itself*:

```sh
pgrep -f "hf download"          # matches the ssh session running the pgrep
pkill  -f "fornax serve"        # kills that ssh session
```

The symptoms are a download that looks as if it is still running long
after it finished, and an ssh session that dies mid-command with exit
255. Two fixes, both cheap:

- **Put the patterns in a script file** on the host and run that. The
  invoking command line then contains only the script's path.
- **Kill by PID**, resolved once, never by pattern:

```sh
PID=$(pgrep -f "$PATTERN" | head -1)   # inside a script, not on the ssh line
kill "$PID"
```

### Reasoning models put their thinking somewhere

Check where before wiring a client to the output.

The Qwen3.8 chat template opens `<think>` **in the prompt**, so the model
generates already inside the block and emits only the closing tag. Raw
`content` therefore begins mid-thought with no opening tag. Nothing is
missing, and nothing is being stripped.

`--reasoning-parser=<name>` moves it to its own field, leaving `content`
as the answer alone:

```
content   : "\n\n1\n2\n3\n4\n5"
reasoning : "We need to respond to user: ..."
```

Streaming splits the same way: deltas carry `reasoning` and `content` as
distinct fields.

**Check the field name against your engine version.** vLLM v0.28.0
calls it `reasoning`; other versions and other engines use
`reasoning_content`. Guessing produces a client that silently reads
nothing.

To skip thinking entirely, the template exposes a switch:

```json
{"chat_template_kwargs": {"enable_thinking": false}}
```

Worth knowing when tokens are expensive: on a slow endpoint, most of
what you wait for is reasoning you did not ask for.

### Streaming is not optional at low token rates

At 3 tok/s, a request that does not stream shows nothing until it is
finished: a 600-token answer is three minutes of blank terminal, which
reads as a hang. The same request streamed feels slow but alive.

Use `curl -N`, and unbuffer whatever parses the stream (`python3 -u`);
either one buffering defeats the other.

## 7. Before committing to a model

1. **Read `config.json`.** The architecture string, the layer types, the
   head geometry. The name tells you nothing: Qwen3.8-27B declares
   `Qwen3_5ForConditionalGeneration`, and Qwen3.8-Flash-Next declares
   `Qwen4ExpForConditionalGeneration`, a different family entirely.
2. **Check the engine's model registry for that exact architecture
   string**, in the version you pin, not in the latest release notes.
   Support for a model published on the day of an engine release is
   usually not in that release.
3. **Do the fit arithmetic** before downloading hundreds of gigabytes.
4. **Do the bytes-per-token arithmetic** before promising anyone a
   latency.
5. **Then measure**, and write the figure down next to the prediction.
   On the models above the fit arithmetic held each time, and the speed
   was the surprise.

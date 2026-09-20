# Migrating to Fornax

Fornax is the new name of llmops. It still prepares model weights, launches
inference engines, and serves healthy endpoints. Lux handles routing between
providers, access policy, and budgets.

## Command and module

Install the renamed command:

```sh
go install latere.ai/x/fornax/cmd/fornax@main
```

Update scripts from `llmops` to `fornax`. The source repository is now
`github.com/latere-ai/fornax`, and its module is `latere.ai/x/fornax`.
GitHub redirects the former repository URL. The old command and environment
variables are not aliases for the new names.

## Names to update

| Previous name | Current name |
| --- | --- |
| `LLMOPS_ENGINE_CMD` | `FORNAX_ENGINE_CMD` |
| `LLMOPS_ENGINE_HEALTH_PATH` | `FORNAX_ENGINE_HEALTH_PATH` |
| `LLMOPS_HF_BASE` | `FORNAX_HF_BASE` |
| `LLMOPS_E2E_SCRATCH` | `FORNAX_E2E_SCRATCH` |
| `LLMOPS_API_KEY` in generated opencode configuration | `FORNAX_API_KEY` |
| `llmops` harness provider ID | `fornax` |
| `X-LLMOps-Compat-Loss`, `X-LLMOps-Speculator` | `X-Fornax-Compat-Loss`, `X-Fornax-Speculator` |
| `llmops_*` Prometheus metrics | `fornax_*` |
| `llmops.*` OTLP instruments and `llmops` service name | `fornax.*` and `fornax` |
| `/usr/local/bin/llmops` | `/usr/local/bin/fornax` |
| `/etc/llmops` | `/etc/fornax` |
| `/var/cache/llmops` in deployment examples | `/var/cache/fornax` |
| `llmops` Kubernetes namespace and application label | `fornax` |
| `llmops-runtime-*`, `llmops-mirror` image names | `fornax-runtime-*`, `fornax-mirror` |

Regenerate coding-harness configuration with `fornax endpoint --harness ...`.
Update monitoring queries and clients that read the project-specific headers.
Model names, service filenames, manifest schemas, S3 object locations, and
checkpoint formats are unchanged.

## Existing installations

No command in this migration moves or deletes model weights. Keep using an
existing cache by passing its path, for example `--cache-root /var/cache/llmops`.
For the local e2e script, set `FORNAX_E2E_SCRATCH` to the former
`~/.cache/llmops-e2e` directory to reuse its downloads.

On a systemd host, review the generated unit before installing it:

```sh
fornax install --manifest models/qwen3.8-27b.yaml \
  --config-dir /etc/llmops --cache-root /var/cache/llmops --print
```

This keeps the existing config and cache paths and points `ExecStart` at
`/usr/local/bin/fornax`. Install the binary at that path, then regenerate the
unit without `--print` when ready. Restart the model service during your chosen
rollout window. Pass the same `--config-dir` to `fornax ps`, `endpoint`, and `run`
until you migrate the configuration directory.

For Kubernetes, build and publish the `fornax-*` images before applying the
updated examples. Choose whether to retain your current namespace and cache
mounts through deployment overrides or migrate them. A new namespace creates
new resources and does not replace the old deployment. If endpoint addresses
change, update their registration in Lux during the rollout.

The repository rename does not publish images or change any running deployment.

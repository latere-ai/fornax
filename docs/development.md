# Development

Building, testing, and changing Fornax itself. To run it, read the
[deploy guide](deploy.md) instead.

## Build

Go 1.27 or newer, no cgo, no other build dependency:

```sh
git clone https://github.com/latere-ai/fornax.git
cd fornax
make build                       # go build ./...
make dist                        # static linux/amd64 and linux/arm64 binaries in dist/
make hooks                       # the pre-commit and pre-push hooks
```

`make dist` stamps the version and commit into the binary; a plain
`go build` still answers `fornax version` from what the toolchain
recorded.

## Test

```sh
make check          # the whole gate, as CI runs it
make test           # go vet and the suite
make cover          # 90% per package, not a repository average
make test-hermetic  # the suite with only the toolchain and system directories on PATH
make test-race      # the suite under the race detector (needs cgo)
make lint           # golangci-lint and the instrumented-HTTP-client check
make spec-lint      # the spec tree against .lateregate.yaml
make validate       # models/*.yaml, their deploy artifacts, the dependency list, and no cgo
make e2e            # pull, push, verify and serve, ready, metrics against fakes
make e2e-local      # the whole pipeline on a laptop with a real model
```

The gates live in [`latere.ai/x/ci-gate`](https://github.com/latere-ai/ci-gate),
pinned as a tool dependency in `go.mod`, so a gate runs the same here as
on a runner. `go tool lateregate list` names them and
`go tool lateregate <name>` runs one. What this repository asserts about
itself, the coverage floor and its exemptions, the spec vocabulary, the
dependency allowlist, is in `.lateregate.yaml`.

**`make cover` gates each package.** An average lets a well-tested
package carry an untested one and reports a number nobody can act on.

**`make test-hermetic` strips `PATH`** to the Go toolchain and the
directories `.lateregate.yaml` names. Tests that relied on whatever
happened to be installed, `systemctl` present on one machine and absent
on another, a coding agent on a laptop's `PATH` and not a runner's, pass
locally and fail in CI. This catches them before a push.

Every target but `e2e-local` needs nothing beyond a Go toolchain and this
checkout, and CI runs them on every push. `make e2e` drives the real code
paths against in-process fakes, so it needs no GPU, no network, and no
credentials. Nothing is skipped for missing configuration.

`make e2e-local` is the one target with outside prerequisites: Docker or
Podman, [uv](https://docs.astral.sh/uv/), Apple silicon, and a one-time
download of about 1.5 GB. It runs the full pipeline against MinIO for
the bucket and a 0.6B model under `mlx_lm`, at no cloud cost.
`FORNAX_E2E_SCRATCH` moves its working directory from
`~/.cache/fornax-e2e`.

GPU serving on real hardware is not covered by any automated suite. It
is a per-model release check, run by hand against the model's spec.

## Layout

```
cmd/fornax/          the one command: weights, serving, harness, bench
internal/
  manifest/          the models/*.yaml schema and its validation
  mirror/            Hugging Face fetch, checksums, freeze, store push and verify
  runtime/           the serving entrypoint: weight prep, preflight, engine launch, the shim
  install/           systemd unit rendering for the bare-metal mode
  harness/           host discovery for `ps`, and agent configuration for `endpoint` and `run`
  deploycheck/       manifest and deploy-artifact consistency
  bench/             the load generator behind `fornax bench`
deploy/              one artifact per model: a LeaderWorkerSet or a unit; the mirror Job
models/              per-model manifests
e2e/local/           the laptop full-pipeline run
specs/               design records; start at specs/README.md
Dockerfile.*         the runtime and mirror images
```

The dialect translation is not here: it is
[`latere.ai/x/pkg/llmdialect`](https://github.com/latere-ai/pkg), shared
with the Lux gateway so the two never disagree about what a request
means.

## Conventions

- Every change carries a test that fails without it.
- Anything larger than a fix starts as a spec in `specs/`, and the spec
  is updated in the same change that lands the work.
- Each document has one reader: `README.md` and `docs/` for people who
  run Fornax or call it, this page and `specs/` for people changing it,
  comments for whoever debugs the code. [CONTRIBUTING.md](../CONTRIBUTING.md)
  says how that applies to messages and errors.
- The manifest schema, the endpoints, and the command flags are not
  frozen yet. A change to any of them updates the deploy guide in the
  same commit, and the changelog under `Unreleased`.

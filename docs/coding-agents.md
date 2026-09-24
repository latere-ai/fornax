# Coding agents

Pointing Claude Code, Codex, or opencode at a model served on a host with
`fornax install`. Every one of them speaks a dialect the endpoint already
serves, so the only work is telling the agent where to look, and three
commands do that.

These commands read what `fornax install` placed on the host: the
manifests in `/etc/fornax` and the units in `/etc/systemd/system`. Run
them on the host itself, or pass `--config-dir` and `--unit-dir` for
another layout.

## See what is serving

```sh
fornax ps
```

```
NAME         STATE  PORT  RUNTIME  GPU     SPECULATOR  LOADED
qwen3.8-27b  ready  8000  vllm     1xgb10  -           39.2s
```

`STATE` is `ready`, `loading`, or `down`, from asking each endpoint
rather than systemd, so a model started by hand shows up too. `LOADED` is
how long the weights took to prepare, and `SPECULATOR` the draft head the
running process was started with. `--json` prints the same as JSON.

## Launch an agent against a model

```sh
fornax run claude
fornax run codex --model qwen3.8-27b -- <codex arguments>
```

`run` sets the variables the agent reads and then becomes the agent, so
signals, exit codes, and the terminal behave exactly as if you had run it
directly. Everything after `--` is passed to the agent. `--model` may be
left out when exactly one model is installed.

A model that is loading or down is refused rather than started: starting
one is a ten-minute weight load nobody asked for. `--wait 15m` blocks on
a model that is already loading until it is ready.

## Or print the configuration

```sh
fornax endpoint --harness claude
fornax endpoint --harness codex > ~/.codex/config.toml
```

`endpoint` prints what the agent reads, for a shell profile, a config
file, or another machine. Pass `--host` with the address the agent will
reach the host on when that is not `127.0.0.1`. `--format env` prints
shell exports for any agent; `--format json` or `--format toml` prints
the agent's config file where it has one.

| Agent | Dialect | What it reads |
|---|---|---|
| `claude` | Anthropic Messages | `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL` |
| `codex` | OpenAI Chat | `OPENAI_BASE_URL`, `OPENAI_API_KEY`, and a `fornax` provider in `~/.codex/config.toml` |
| `opencode` | OpenAI Chat | `FORNAX_API_KEY`, and a `fornax` provider in `opencode.json`, since opencode has no variable for the base URL |

Codex and opencode read a config file as well as the environment, so
`fornax run codex` and `fornax run opencode` remind you to write it
first with `fornax endpoint`.

## The token is a placeholder

The endpoint checks no credential. The token these commands emit,
`local` unless you pass `--token`, exists only so an agent does not fall
back to its own sign-in flow. It protects nothing; keep the port off
networks you do not trust, as the [deploy guide](deploy.md#security)
describes.

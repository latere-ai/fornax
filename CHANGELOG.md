# Changelog

Every tag has a section here, and the section is the body of the GitHub
release. A tag without one is refused at the pre-push and fails the release
workflow. Write under `Unreleased` as work lands; `lateregate release vX.Y.Z`
turns that into the tag's section, commits, tags and pushes.

A section says what changed for whoever uses the release, not what was
committed: the commit log already holds that.

## Unreleased

- Rename the project to Fornax, its repository to `latere-ai/fornax`, and its
  Go module to `latere.ai/x/fornax`. The command is now `cmd/fornax`.
- Update configuration, headers, metrics, harness provider names, image names,
  and deployment defaults to Fornax. Model IDs and weight formats are unchanged.

### Fixed

- OpenTelemetry Go v1.46.0, with the log modules at v0.22.0, the slog bridge at
  v0.20.1 and otelhttp at v0.71.0, past GO-2026-6615 and GO-2026-6505, and
  `latere.ai/x/pkg` v0.90.2. pkg v0.90.2 names the service resource with
  semantic conventions v1.43.0, the schema of this SDK; with an older schema
  the two conflict when merged and `fornax serve` disables telemetry export at start.

- The deploy artifacts no longer name `ghcr.io/latere-ai/fornax-*:v0.1.0`,
  images that were never published. They carry the placeholder tag
  `unreleased`, and the deploy guide gives the one `sed` command that
  replaces it with the registry and version you pushed.

### Security

- Built with Go 1.27.2 and golang.org/x/net v0.60.0, which fix GO-2026-6611, GO-2026-6612, GO-2026-6613 and GO-2026-6617.

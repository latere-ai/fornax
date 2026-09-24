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

- The deploy artifacts no longer name `ghcr.io/latere-ai/fornax-*:v0.1.0`,
  images that were never published. They carry the placeholder tag
  `unreleased`, and the deploy guide gives the one `sed` command that
  replaces it with the registry and version you pushed.

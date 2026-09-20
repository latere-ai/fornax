---
title: "Rename the model deployment and serving component to Fornax"
status: draft
depends_on:
  - 024-single-cli.md
affects:
  - go.mod
  - cmd/llmops/
  - internal/
  - deploy/
  - models/
  - docs/
  - Makefile
effort: medium
created: 2026-09-20
updated: 2026-09-20
author: changkun
dispatched_task_id: null
---

# Fornax naming migration

Fornax is the name of the model deployment and serving component previously
called llmops. It prepares weights, launches inference engines, and exposes
healthy model endpoints. Lux retains provider routing, access, and budget policy.

## Scope and acceptance criteria

1. Rename the GitHub repository in place to `latere-ai/fornax`, preserving its
   history, issues, and tags. Use `latere.ai/x/fornax` as its Go module and move
   `cmd/llmops` to `cmd/fornax`.
2. Update imports, CLI output, build recipes, CI configuration, docs, and specs.
   Use `FORNAX_*` environment variables, `fornax_*` Prometheus names,
   `fornax.*` OTLP names, `X-Fornax-*` headers, and `fornax` harness provider IDs.
3. Update deployment examples together: binary and config paths, container image
   names, namespace, and cache defaults. Preserve model IDs, weight formats,
   model-specific service names, and S3 object locations. Document how to reuse
   existing paths and update a deployment deliberately.
4. Update sibling project references. Preserve unrelated work, historical
   snapshots, and external citations using the general term LLMOps.
5. Verify the existing Go tests and e2e suite, coverage gate, model/deployment
   consistency, build, formatting, spec and dependency checks, and configured
   distribution builds. Verify the built CLI and installation from the published
   vanity module. No name-only tests are added.

## Deployment boundary

This migration changes source and examples. It does not apply Kubernetes
manifests, restart services, move model caches, publish container images, or
create a release tag. An operator builds the new images and chooses the rollout
before applying renamed deployment examples.

## Decisions

- One current name across command, module, and configuration avoids a permanent
  split between product branding and the interfaces developers use.
- Existing installations remain under operator control. A naming change never
  implicitly moves weights or creates a second live deployment.

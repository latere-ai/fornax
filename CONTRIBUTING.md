# Contributing

Issues and pull requests are welcome. [docs/development.md](docs/development.md)
covers building, the test targets, and the repository layout;
[specs/](specs/README.md) holds the design record behind each decision.

## Sending a change

- Run `make check` before you push. It is the gate CI runs, and
  `make hooks` installs the pre-commit and pre-push hooks that run parts
  of it earlier.
- Every bug fix ships with a test that fails without it.
- Keep each commit one small, logical change, with a message that says
  why.
- Anything larger than a fix, and anything that changes the manifest
  schema, an endpoint, or a command flag, starts as a spec in `specs/`,
  so the reasoning arrives with the change. Open an issue first if you
  are unsure.
- A change a user would notice adds a line under `Unreleased` in
  [CHANGELOG.md](CHANGELOG.md), and updates the page in `docs/` that
  describes it.

## Writing

Every sentence Fornax emits or carries is written for one reader, and the
register follows the reader:

- User, a person or a coding harness: CLI output, the endpoint's error
  `message`, the docs. Short and plain: what happened and what to do
  next, naming a command or a page, never a package, a function, a
  table, or a Kubernetes object.
- Contributor, someone changing Fornax: specs, this file, package
  documentation, commit messages, source comments. Precise, in the
  project's own terms, with the reason a design is what it is.
- Developer, someone debugging a running system: logs, traces, serving
  and mirror failures, startup errors. Exact and complete: object,
  operation, observed value, expected value, and the underlying error.

An error has one code, one fixed user sentence in `message`, and one
developer detail in a separate field shown only on request. The canonical
statement, worked examples, and the review checklist are in the
[registers document](https://github.com/latere-ai/pkg/blob/main/docs/writing/registers.md)
in pkg. The rule applies to new text and to reviews; existing text is
fixed as it is touched.

# ADR-0007 — The engine repo owns example/test workflows; production workflows live in external repos

- **Date:** 2026-06-07
- **Status:** Accepted

## Context

"Workflow packages are committed source; run instances are not" (CONTEXT,
ADR-0002 territory) settles *what* is versioned, but not *which repo* a
workflow package lives in. In practice there are two populations with
different ownership:

- **Example/test workflows** the engine needs to prove and regression-test
  itself — small, generic, and meaningful only as engine fixtures (this
  feature's tiny script-plus-gate smoke is the first).
- **Production workflows** (the fan-out researcher, the labs-video
  pipeline, dev-skills replacements) that carry real creative context,
  fixtures, brand constraints, and package-local skills. The deprecated
  prototype already treated these as external — it consumed a workflow
  package by workspace-dir/path and explicitly "did not own its creative
  context" — but that boundary was never written into the new docs.

Leaving it implicit risks two failure modes: production creative assets
leaking into the engine repo, or the engine growing coupling to a specific
production workflow's domain.

## Decision

Split workflow-package ownership by repo:

- **The engine repo owns example/test workflow packages**, committed under
  `workflows/` (e.g. `workflows/tiny-smoke/workflow.yaml`). They exist to
  prove and integration-test the engine and double as runnable
  documentation.
- **Production workflow packages live in their own external repos.** The
  engine **consumes** them by path/reference and runs them, but never owns
  their creative context, fixtures, or package-local skills. The consume
  seam (a workspace/package path resolved at `run create`) is the same one
  the prototype used.
- **Run instances stay gitignored** under `runs/<id>/` regardless of which
  repo the package came from (unchanged from the committed-vs-gitignored
  stance).

The explicit *no* — the engine does not own production creative context —
is the load-bearing half: it keeps the engine a general runner rather than
a home for any one workflow's domain.

## Consequences

- This feature's smoke workflow ships in-repo at `workflows/tiny-smoke/`.
- The CLI/loader must accept a workflow package path that may resolve
  outside the engine repo; in-repo `workflows/` is just the default search
  location, not the only one.
- When the first external production workflow lands, no doc change is
  needed — it's already the expected shape.

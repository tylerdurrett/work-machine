# ADR-0002 — Repo owns canonical engineering docs

- **Date:** 2026-06-06
- **Status:** Accepted

## Context

The project began with orientation docs in the Labs Obsidian vault. That was useful for shaping the idea, but Work Machine is now a real repository with code, agent workflow docs, ADRs, and an issue tracker.

Keeping detailed architecture, roadmap, and implementation docs in both the vault and repo would create drift. Agents working in the repo also need code-adjacent context available without depending on a private vault path.

## Decision

The Work Machine repo owns canonical engineering documentation:

- `README.md` for the project entry point;
- `CONTEXT.md` for vocabulary and domain context;
- `docs/charter.md` for mission, principles, scope, and non-goals;
- `docs/architecture.md` for canonical system shape;
- `docs/north-star.md` for vision and direction;
- `docs/roadmap.md` for repo-level phase sequencing before work becomes issues;
- `docs/adr/` for accepted technical decisions;
- GitHub issues for decomposed specs and implementation tasks.

The Labs Obsidian vault owns only the basic project gist, orientation links, and durable Labs-level decisions.

## Consequences

**Positive:**

- Agents and humans can work from the repo alone.
- Repo docs, tests, examples, ADRs, and issues evolve together.
- The vault stays lightweight and does not compete with the implementation control plane.

**Costs:**

- Vault project docs must be trimmed to pointers after migrations.
- If a strategic decision starts in the vault, it should be promoted into a repo ADR once it affects implementation.

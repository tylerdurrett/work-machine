# Work Machine Charter

## Mission

Build a small, understandable workflow system that coordinates agent, script, worker, artifact, tracker, and human-review steps through adapter-based issue/card surfaces.

## Core idea

A serious workflow run has:

```text
workflow definition + coordinator state + tracker surface + executors + workers + artifacts + gates
```

The tracker is where humans see and command the work. The coordinator decides valid transitions. Workers execute steps. Artifact storage holds the files.

## First audience

Tyler / Labs, using repo-native development and GitHub Issues as the first concrete tracker surface.

## Design principles

- **Repo first for engineering truth.** Use repo docs, tests, examples, ADRs, and issue tracker items for implementation work.
- **Vault for orientation.** Keep the Labs vault folder focused on purpose, links, durable decisions, and current phase.
- **Fat engine, thin skill.** Deterministic code owns state, transitions, artifacts, gates, and validation. Agents own judgment-heavy steps.
- **Tracker as surface, not worker.** GitHub Issues/Trello cards are human interfaces, not execution engines.
- **Artifacts are first-class.** Videos, documents, generated files, hashes, and review URLs belong in an artifact index and storage adapter.
- **Start local, keep cloud-shaped seams.** Local worker first; local/cloud workers later.
- **Do not prematurely choose a heavyweight runtime.** Keep Temporal/Semaphore-style durability in view, but first prove product shape.

## Non-goals for the first slice

- Full Temporal integration.
- Full Trello adapter.
- Full R2/S3/Frame.io integration.
- Generic marketplace of workflows.
- Complex multi-worker scheduling.
- Replacing the existing Hermes Kanban experiment immediately.

## Success criteria for the first slice

A tiny workflow can be started, tracked, executed, reviewed, and completed through an issue/card tracker surface while preserving machine-readable run state and artifact metadata.

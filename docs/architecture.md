# Work Machine Architecture

## Working thesis

Work Machine should begin as a custom coordinator with clean tracker-adapter seams. GitHub Issues is the first serious tracker adapter, not the whole architecture.

```text
workflow.yaml
  -> coordinator / run state
  -> tracker adapter: GitHub Issues first, Trello/other surfaces later
  -> executor adapters: script, agent, human/manual
  -> workers: local Mac first, cloud later
  -> artifact storage: local filesystem first, R2/S3/Frame.io later
  -> notification adapter: GitHub comments / Discord
```

## Core components

### Workflow package

A workflow package contains declarative workflow structure and supporting assets:

```text
workflow.yaml
roles.yaml              # later, if useful
artifact-policies.yaml  # later, if useful
skills/                 # optional package-local procedures
validators/             # optional artifact validators
examples/
```

### Coordinator

The coordinator owns valid transitions for a run. In the first version it can be lightweight and local:

```text
runs/<run_id>/
  run.yaml
  events.jsonl
  artifact-index.yaml
  workflow.snapshot.yaml
```

Early tracker labels/comments/fields may drive behavior, but the run directory should exist from day one so the system does not become purely stringly-typed tracker automation.

### Tracker adapter

GitHub Issues is the first concrete tracker adapter. More generally, an issue/card is the visible run surface:

- run metadata;
- current step/status;
- artifact links;
- Mermaid workflow graph;
- allowed commands such as `/approve` or `/request-changes`;
- discussion and decision history.

Longer-term, the tracker is a projection and command surface. The coordinator is canonical.

### Executor adapters

Initial executor types:

- `script` — deterministic local command;
- `agent` — Hermes or other LLM agent step;
- `human` — manual task or review gate;
- `noop` — tracking-only placeholder for early tests.

Later executor types may include webhook, queue, render, or subworkflow.

### Workers

A worker actually executes steps. Start with a local worker process that can run scripts and invoke local agents. Later workers can advertise capabilities and run on cloud/GPU boxes.

### Artifact storage

Start with local filesystem artifacts and an artifact index. Do not store large video payloads in GitHub. The GitHub issue should link to artifacts exposed by the storage adapter.

Future adapter classes:

- `local_fs` for development;
- `r2` / `s3` for large durable blobs;
- `frameio` for video review and timecoded comments.

### Gates

A gate is a state-machine wait, not just a comment convention.

Initial issue-tracker flow:

```text
coordinator opens gate
  -> issue/card comment asks for /approve or /request-changes
  -> poller reads tracker comments/events
  -> coordinator validates command
  -> event is appended
  -> tracker projection is updated
```

## First vertical slice

The first slice should prove one complete run, not every adapter:

1. Load a tiny workflow definition.
2. Create a local run directory and event log.
3. Create a GitHub issue with run metadata.
4. Execute one local script step.
5. Record one artifact path/hash in `artifact-index.yaml`.
6. Update the issue with artifact links.
7. Open one human approval gate.
8. Accept `/approve` from the issue.
9. Complete the run.

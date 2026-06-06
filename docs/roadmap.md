# Work Machine Roadmap

Detailed implementation work should live in GitHub issues. This document keeps the repo-level phase map before work is decomposed into specs.

## Phase 0 — Project setup / docs migration

- [x] Create repo: `tylerdurrett/work-machine`.
- [x] Establish repo as canonical engineering docs surface.
- [x] Create project-facing README, context, charter, architecture, and roadmap docs.
- [ ] Add basic package skeleton.
- [ ] Open tracker issues for the first vertical slice.

## Phase 1 — Tiny tracker-backed run

Goal: prove an issue/card tracker surface plus local run state.

- [ ] Define tiny `workflow.yaml` schema.
- [ ] Create run directory with `run.yaml`, `events.jsonl`, `artifact-index.yaml`, `workflow.snapshot.yaml`.
- [ ] Create tracker issue/card for a run.
- [ ] Render run status and allowed commands into issue/card body or comments.
- [ ] Implement local script executor.
- [ ] Record artifact path, size, and hash.
- [ ] Open human gate and parse `/approve` from tracker comments/events.
- [ ] Complete run and update issue.

## Phase 2 — Agent step and artifact contracts

- [ ] Add `agent` executor adapter.
- [ ] Support declared consumes/produces artifact contracts.
- [ ] Add deterministic artifact validation.
- [ ] Add failure/blocking states.
- [ ] Add basic reconciliation between issue projection and run state.

## Phase 3 — Real workflow package smoke

- [ ] Create one useful Labs workflow package.
- [ ] Include one agent step, one script/render step, and one gate.
- [ ] Keep local filesystem artifact storage first.
- [ ] Decide whether R2/S3 or Frame.io deserves the next spike.

## Phase 4 — Runtime/substrate evaluation

Only after the custom coordinator hits real pain, evaluate:

- Temporal for durable coordinator semantics;
- Hatchet/Inngest/Trigger-style systems for execution/worker routing;
- Mastra/LangGraph for individual agent-step implementation and evals.

# Work Machine Research Notes

These notes are background research and architecture exploration for Work Machine. They are not implementation specs; GitHub issues own decomposed work.

## Notes

- [Tracker-Agnostic Workflow Architecture](tracker-agnostic-workflow-architecture.md) — architecture pivot from Hermes Kanban toward tracker/executor/artifact adapters.
- [Workflow Orchestration Candidates](workflow-orchestration-candidates.md) — due-diligence pass across Temporal, LangGraph, Mastra, Hatchet, Windmill, Frame.io, R2/S3, and adjacent tools.
- [Coordinator and Tracker State Architecture](coordinator-and-tracker-state-architecture.md) — coordinator-vs-tracker state model, custom coordinator path, Temporal path, workers, gates, and reconciliation.

## Canonical docs relationship

Use these notes as source material and design memory. Promote stable decisions into [ADRs](../adr/) and implementation-facing contracts into canonical docs such as [architecture](../architecture.md), [roadmap](../roadmap.md), or future schema/adapter docs.

# Work Machine Context

## One-sentence definition

Work Machine coordinates tracker-backed workflow runs across human review, agent work, scripts, workers, artifacts, and gates.

## Core vocabulary

- **Workflow package** — declarative workflow definition plus any supporting assets, validators, examples, or package-local procedures.
- **Run** — one execution of a workflow package, with machine-readable state and event history.
- **Coordinator** — deterministic engine that owns run state, validates transitions, and decides what is allowed next.
- **Tracker adapter** — integration that projects run state into an issue/card surface and reads human commands back from that surface.
- **Tracker surface** — the human-visible issue/card where status, artifacts, discussion, and commands live.
- **Executor adapter** — implementation of a step type such as `script`, `agent`, `human`, or `noop`.
- **Worker** — process or machine that executes runnable steps. Starts local; may become cloud/GPU-capability-aware later.
- **Artifact index** — machine-readable metadata for outputs such as file paths, sizes, hashes, URLs, and validation state.
- **Gate** — a coordinator-owned wait state that requires an external signal, commonly a human tracker command like `/approve`.
- **Projection** — the tracker-facing rendering of coordinator state. The projection is useful, but not canonical.

## Language rules

- Say **tracker adapter** when referring to GitHub Issues, Trello, or future issue/card integrations.
- Say **coordinator** for the deterministic state machine. Avoid making GitHub Issues sound canonical.
- Say **artifact index** for metadata. Avoid implying large artifacts live in GitHub.
- Say **gate** for state-machine waits. Avoid treating `/approve` as just a comment convention.

## First concrete choices

- First tracker adapter: GitHub Issues.
- First worker: local process on Tyler's machine.
- First artifact backend: local filesystem plus artifact index.
- First workflow proof: one tiny script step plus one human approval gate.

## Out of scope for the first slice

- Temporal or another heavyweight workflow runtime.
- Trello adapter.
- R2/S3/Frame.io artifact storage.
- Complex multi-worker scheduling.
- Generic workflow marketplace.

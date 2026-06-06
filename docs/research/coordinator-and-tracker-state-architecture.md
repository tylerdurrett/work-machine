# Coordinator and Tracker State Architecture

> Research/background copied from the Labs vault on 2026-06-06 as starting context for Work Machine. Preserve this as research; repo issues own implementation tasks.

This note follows [Workflow Orchestration Candidates](workflow-orchestration-candidates.md) and [Tracker Agnostic Workflow Architecture](tracker-agnostic-workflow-architecture.md). It narrows the discussion from candidate products to a concrete architecture question:

> Where does workflow state live, what runs individual steps, and how does the human-visible issue/card surface relate to the actual coordinator?

The current project started as a Hermes Kanban experiment. The next architecture should stay grounded in that learning while opening a path toward a more robust system with tracker adapters, executor adapters, artifact storage adapters, and human review gates.

## Current position

The existing `workflow-engine` has already proven useful building blocks:

- `workflow.yaml` as a concise workflow definition;
- deterministic validation and planning;
- run manifests;
- artifact contracts;
- gate metadata;
- approval policies;
- dry-run inspection;
- Kanban materialization;
- the “fat engine, thin skill” principle.

But Hermes Kanban is not the right long-term human/team surface, and it is not a general deterministic script executor. The next version should separate these concerns:

```text
workflow definition
  -> coordinator/state machine
  -> tracker surface
  -> executor workers
  -> artifact storage/review
  -> notification/gate commands
```

## Full layer picture

A mature version has these layers:

```text
Human surfaces
  Discord, GitHub Issues/Projects, Trello, Frame.io

Workflow Engine API
  create runs, receive webhooks, parse commands, signal coordinator

Coordinator / state machine
  custom lightweight coordinator or Temporal

Executor adapters
  script, agent, render, webhook, queue, human/manual

Workers
  local Mac mini, cloud CPU worker, GPU/render worker, agent service

Artifact and review storage
  local filesystem, R2/S3, Frame.io
```

The important design principle:

> The tracker is where humans see and command the workflow. The coordinator is what decides valid transitions. Workers do the work. Artifact stores hold the files.

In an early prototype, the tracker may temporarily be the canonical workflow state. In a more robust system, the coordinator should become canonical and the tracker should become a projection plus command surface.

## What runs individual steps?

Even with Temporal or another coordinator, individual steps are not “run by GitHub” or “run by the issue.” They are run by worker processes.

A worker can run anywhere:

- local Mac mini;
- cloud VM;
- GPU box;
- container worker;
- app server;
- agent service.

A step might be executed by:

- a local script;
- a Hermes profile/skill;
- a render command such as Remotion + ffmpeg;
- an API call;
- a queue job;
- a human manual action;
- eventually a Mastra/LangGraph/OpenAI/PydanticAI agent implementation for a specific step.

Example worker responsibilities:

```text
receive step assignment
load workflow contract
pull required artifacts
run script/agent/render
validate outputs
upload artifacts
return artifact metadata
update tracker or signal coordinator
```

For video workflows, a local Mac mini worker is plausible and useful. It can own steps that need local files, local credentials, Hermes access, Remotion, ffmpeg, or large-media tooling. Cloud workers can later handle generic scripts, API calls, or heavier GPU/container workloads.

## Version A: custom coordinator path

The custom coordinator path is the most grounded next step from the current repo. It keeps the existing `workflow.yaml` and run-manifest ideas, but shifts the external surface from Hermes Kanban to GitHub Issues or another tracker.

### Shape

```text
workflow.yaml
  -> workflow-engine planner
  -> run.yaml / events.jsonl / artifact-index.yaml
  -> GitHub Issue tracker adapter
  -> local worker executes steps
  -> R2/S3/local artifacts
  -> Discord/GitHub human gates
```

### State files

A lightweight custom coordinator might use:

```text
runs/<run_id>/
  run.yaml
  events.jsonl
  artifact-index.yaml
  workflow.snapshot.yaml
```

`run.yaml` stores current machine-readable state. `events.jsonl` records transitions and decisions. `artifact-index.yaml` records artifact URIs, hashes, metadata, and review links. `workflow.snapshot.yaml` captures the exact workflow version used for the run.

### Step execution

A local worker could poll or reconcile runs:

```bash
workflow-engine worker run \
  --tracker github \
  --repo owner/repo \
  --capabilities script,agent,remotion,ffmpeg,local-files
```

The worker loop:

```text
1. find ready runs or tracker issues
2. claim a step
3. load workflow.yaml and run state
4. verify inputs/artifacts
5. execute the step
6. validate outputs
7. write artifact index
8. append event
9. update tracker labels/comments/body
10. notify if a gate opens
```

### Strengths

- Fastest path from current repo.
- Preserves Labs momentum.
- Easy to inspect and debug.
- Keeps `workflow.yaml` authoring central.
- Lets GitHub/Trello UX be tested before committing to Temporal.
- Forces the right adapter abstractions.

### Weaknesses

A custom coordinator will eventually need to solve:

- durable waits;
- retries;
- idempotency;
- concurrent worker claims;
- crash recovery;
- event history;
- timers/escalations;
- workflow versioning;
- partial tracker update failures.

That is acceptable for a prototype if the design keeps a migration path open.

## Version B: Temporal-centered path

Temporal can be used as the durable macro-coordinator. It owns the long-running workflow state, event history, timers, retries, signals, and activity dispatch.

### Shape

```text
workflow.yaml or workflow package
  -> workflow-engine compiler/interpreter
  -> Temporal workflow execution
  -> Temporal activities on task queues
  -> workers on local/cloud machines
  -> R2/S3/Frame.io artifacts
  -> GitHub/Trello projection
  -> Discord/GitHub/Frame.io signals for gates
```

### Temporal concept map

| Workflow-engine concept | Temporal concept |
|---|---|
| workflow run | workflow execution |
| step | activity or child workflow |
| deterministic coordinator | workflow function/history |
| external side effect | activity |
| human approval | signal |
| gate wait | workflow waiting for signal/condition |
| local/cloud worker | Temporal worker on task queue |
| subworkflow | child workflow |
| retry policy | activity retry options |
| timeout/deadline | timers/activity timeouts |
| run state | Temporal history + compact workflow state |

### Workers and task queues

Temporal workers subscribe to task queues. A local Mac mini worker could listen for local render tasks:

```text
Task queue: local-render
Capabilities: remotion, ffmpeg, local files, Hermes
```

A cloud worker could listen for generic API/script tasks:

```text
Task queue: cloud-general
Capabilities: python, node, http, artifact-upload
```

An agent worker could listen for agentic steps:

```text
Task queue: agent-steps
Capabilities: hermes-agent, model-calls, structured-output
```

The Temporal workflow schedules an activity to the right queue:

```text
render_draft -> task_queue=local-render
post_github_update -> task_queue=tracker-updates
upload_artifact -> task_queue=artifact-storage
```

### Human gates

A gate is represented as a durable wait:

```text
Temporal workflow opens gate
  -> activity posts GitHub comment and Discord notification
  -> workflow waits for signal
  -> GitHub/Discord/Frame.io webhook sends approval signal
  -> workflow validates decision and advances
```

A GitHub `/approve` comment is not itself canonical truth. It is a command that becomes canonical only after the workflow receives and validates the signal.

### Strengths

- Strong correctness and durability.
- Long-running human gates are natural.
- Timers, retries, and failure recovery are built in.
- Local/cloud workers are first-class.
- Subworkflows are first-class.
- Event history is durable.

### Weaknesses

- More infrastructure and conceptual overhead.
- Workflow implementation is code-first unless a `workflow.yaml` interpreter/compiler is built.
- Temporal determinism constraints affect implementation style.
- Tracker, artifact, review, and workflow-authoring UX still need to be built.

### What Temporal does not replace

Temporal does not replace:

- GitHub/Trello as human-visible tracker;
- R2/S3/Frame.io as artifact storage/review;
- Hermes/Mastra/LangGraph/OpenAI/PydanticAI as agent-step implementation choices;
- workflow authoring layer;
- tracker adapter logic.

It replaces or upgrades the custom coordinator/runtime layer.

## Tracker-as-state vs coordinator-as-state

This is the central design question.

### Model 1: tracker as canonical state

In this model, GitHub/Trello labels, fields, columns, and comments are the actual workflow state.

Example issue labels:

```text
workflow:labs-video
run:active
step:storyboard
status:needs-human-review
gate:storyboard-review
```

A worker watches GitHub:

```text
find issues with workflow:labs-video + status:ready
claim issue
run current step
update labels/comments/artifacts
```

Advantages:

- GitHub/Trello is the visible source of truth.
- Manual issue/card creation can work naturally.
- Teammates can inspect and manipulate state.
- Workers can be simple pollers.
- No hidden coordinator state.
- Fastest to prototype.

Disadvantages:

- Labels/fields are stringly typed and coarse.
- Race conditions require careful claiming semantics.
- Crash recovery gets tricky.
- Complex branching and retries become messy.
- Audit history is fragmented across comments/events.
- Artifact truth should still live elsewhere.
- Tracker APIs are not designed to be durable workflow engines.

This model is probably good enough for an early GitHub-first prototype.

### Model 2: coordinator as canonical state

In this model, a custom coordinator or Temporal stores canonical workflow state. GitHub/Trello is a projection and command surface.

Canonical state includes:

- current step;
- status;
- open gate;
- allowed decisions;
- workflow version;
- artifact IDs and hashes;
- retry counts;
- branch path;
- event history.

Projected tracker state includes:

- labels;
- project status;
- issue/card body;
- comments;
- links to artifacts;
- visible command instructions.

A GitHub issue might say:

```text
status:needs-review
step:storyboard
```

But the machine trusts the coordinator state first.

Advantages:

- Stronger state-machine semantics.
- Cleaner retries and idempotency.
- Better recovery.
- Better artifact provenance.
- Better support for local/cloud workers.
- Better long-running gates and subworkflows.

Disadvantages:

- Tracker can temporarily diverge from canonical state.
- Manual tracker edits need a reconciliation/command path.
- Manually created issues are not runs until an intake process creates/attaches a run.
- More moving parts.
- The system can feel less transparent unless projections are excellent.

### Model 3: tracker as command surface, coordinator as canonical state

This is the likely serious end-state.

```text
Coordinator state = canonical
GitHub/Trello = human-facing projection + command input
```

Example flow:

```text
1. Coordinator opens storyboard review gate.
2. GitHub issue gets labels/status/comment/artifact links.
3. Tyler comments `/approve`.
4. Webhook or poller sends command to coordinator.
5. Coordinator validates: correct gate, allowed decision, authorized user, required artifacts exist.
6. Coordinator records decision.
7. Coordinator updates GitHub labels/comments and advances.
```

In this model, GitHub is not ignored. It is an official workflow interface. But manual changes become commands/events, not automatically trusted facts.

## Manually created tracker issues/cards

If the coordinator is canonical, a manually created GitHub issue or Trello card will not automatically be a workflow run unless an intake mechanism creates or attaches one.

That can be solved deliberately.

### Intake option A: issue template request

A manually created issue includes structured workflow request metadata:

```yaml
workflow: labs-video
input:
  source_url: https://...
```

or labels:

```text
workflow:labs-video
status:requested
```

A watcher sees it:

```text
GitHub issue created
  -> intake watcher detects workflow request
  -> coordinator creates run
  -> coordinator writes run_id back to issue
  -> issue becomes governed by coordinator
```

### Intake option B: chat creates run and issue

This is cleaner:

```text
Tyler chats request
  -> agent/coordinator creates workflow run
  -> coordinator creates GitHub issue/card
```

The tracker issue is born attached to a run.

### Intake option C: reconcile orphan workflow issues

A periodic process finds issues that look like workflow requests but lack a run ID:

```text
label: workflow:labs-video
missing run:<id>
```

It can:

- create a run;
- ask for missing input;
- mark invalid.

## Divergence and reconciliation

If tracker and coordinator state both exist, divergence is inevitable unless managed.

Examples:

- Coordinator says `rendering`, GitHub still says `needs-review` because label update failed.
- Someone manually applies `approved`, but the coordinator is not waiting at that gate.
- GitHub links to an artifact whose hash does not match the artifact index.
- Worker produced an artifact but crashed before updating the tracker.

The system needs a reconciliation loop:

```text
read canonical run state
read tracker state
compare
repair projection or flag conflict
```

Rules:

- The artifact index is canonical for files and hashes.
- The coordinator is canonical for workflow transitions once introduced.
- Tracker comments/labels are commands or projections.
- Invalid manual changes should be reverted or commented on, not silently trusted.
- Every tracker issue/card should include a `run_id` once attached.

## Agent framework role: Mastra and LangGraph

Mastra and LangGraph should not be the starting point for the whole system unless the project intentionally becomes an AI-framework-first build.

The concern is valid:

> Avoid a state machine inside a state machine unless the boundary is clear.

If Temporal or a custom workflow coordinator owns macro workflow state, then LangGraph/Mastra should not independently own the same workflow lifecycle. That creates duplicated state, confusing recovery, and unclear gate ownership.

A safer role:

```text
Coordinator owns macro workflow:
  intake -> brief -> review gate -> storyboard -> render -> review gate

Mastra/LangGraph optionally owns one agent step:
  generate brief by doing research -> draft -> critique -> revise -> structured output
```

In this model, Mastra/LangGraph is an implementation detail of an `agent` executor adapter. It can provide:

- eval tools;
- traces;
- structured step experiments;
- prompt/tool iteration;
- internal agent reasoning graphs;
- step-level quality improvement.

But the workflow run still sees only:

```text
agent step started
agent step produced artifact
agent step failed
agent step returned structured output
```

This keeps the macro state machine simple.

### When Mastra/LangGraph might replace the coordinator

They might be viable as the top-level runtime only if workflows are primarily agent reasoning graphs with light external tools. That does not currently seem like the best starting assumption because the desired system includes:

- scripts;
- rendering;
- local/cloud workers;
- large artifacts;
- GitHub/Trello tracker state;
- human/team gates;
- storage/review adapters.

So the grounded plan is:

> Do not start with Mastra or LangGraph as the main coordinator. Revisit them for individual agent steps, evals, and step-hardening experiments.

## Workflow authoring requirement

A major design requirement is that workflows should be easy to spin up using an agent.

That means the workflow definition format should be:

- expressive enough for inputs, outputs, artifacts, gates, executors, and validation;
- concise enough for humans and agents to draft;
- deterministic enough for validation and compilation;
- visualizable;
- portable across coordinator backends where possible.

Even in a Temporal-backed future, it may be valuable to keep a declarative workflow package:

```text
workflow.yaml
roles.yaml
artifact-policies.yaml
skills/
templates/
validators/
```

Temporal does not require abandoning `workflow.yaml`. Possible strategies:

1. `workflow.yaml` drives a generic Temporal interpreter workflow.
2. `workflow.yaml` compiles into generated Temporal workflow code.
3. `workflow.yaml` remains the authoring/project surface, while hand-written Temporal code implements stable workflow families.

The first option is most flexible for agent-authored workflows. The second can be more robust but harder to iterate. The third may be suitable once a workflow family stabilizes.

## Grounded migration path

A reasonable path from where the project is now:

### Phase 1: GitHub-first custom coordinator prototype

Goal: test the human/team surface and workflow authoring model.

Build:

- GitHub Issues tracker adapter;
- label/comment schema;
- `run_id` attached to issues;
- local run manifest and event log;
- artifact index;
- local script executor;
- Hermes agent executor;
- local filesystem artifact adapter;
- human review gate via GitHub comments and/or Discord;
- basic reconciler.

In this phase, GitHub may be close to canonical state, but the artifact index and event log should already exist so migration is possible.

### Phase 2: coordinator-owned state

Goal: move from label-driven behavior to structured run-state behavior.

Build:

- `run.yaml` becomes canonical for workflow state;
- GitHub becomes projection + command surface;
- worker reads coordinator state rather than only labels;
- tracker reconciliation repairs drift;
- manual issue intake creates/attaches runs;
- R2 artifact adapter spike;
- Frame.io review adapter spike if video review requires it.

### Phase 3: evaluate Temporal migration

Goal: decide whether custom coordinator is hitting durable runtime complexity.

Evaluate:

- long-running human gates;
- retries/timeouts;
- worker crash recovery;
- concurrent local/cloud workers;
- subworkflows;
- event history;
- workflow versioning;
- operational observability.

If these become central, implement a Temporal-backed coordinator spike.

### Phase 4: Temporal-backed serious runtime, if justified

Goal: replace custom runtime machinery with Temporal where it earns its keep.

Build:

- generic Temporal workflow interpreter or compiled workflow family;
- Temporal activity workers for script, agent, render, tracker, artifact, notification steps;
- GitHub/Trello projection adapters;
- signal handlers for GitHub/Discord/Frame.io commands;
- local Mac mini worker and at least one cloud/general worker;
- R2/S3 artifact storage as canonical large-object backend.

## Recommended near-term stance

Do not prematurely choose Temporal, Hatchet, Mastra, or LangGraph as the whole answer. The most useful next build is a small but real GitHub-first coordinator that proves:

- issues/cards feel better than Hermes Kanban;
- workflows can be agent-authored in a concise config;
- local workers can run deterministic scripts;
- agent steps can produce declared artifacts;
- review gates can be handled through tracker comments and Discord;
- artifacts can be indexed and linked cleanly;
- the system can reconcile visible tracker state with run state.

Keep the architecture Temporal-compatible by using the same conceptual boundaries:

```text
workflow definition
coordinator state
activities/executors
signals/commands
task queues/workers
artifact index
tracker projection
```

That way the prototype teaches the product shape without closing off the robust runtime path.

## Working conclusion

There are two viable mental models for the next phase:

1. **GitHub-first prototype:** GitHub issue/card state is highly visible and may initially drive worker behavior.
2. **Coordinator-first serious system:** custom coordinator or Temporal owns canonical state; GitHub/Trello is the projection and command surface.

The project can intentionally move from 1 to 2.

The important part is not to confuse the roles:

```text
GitHub/Trello: human visibility and commands
Coordinator: valid state transitions
Workers: actual execution
R2/S3/local/Frame.io: artifacts and review media
Discord/email: notifications and lightweight responses
Mastra/LangGraph: optional agent-step implementation/eval tools, not the starting macro-coordinator
```

This keeps the workflow-engine grounded in the current experiment while giving it a credible path to a more robust, artifact-heavy, human-gated workflow system.

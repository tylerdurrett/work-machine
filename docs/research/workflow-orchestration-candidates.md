# Workflow Orchestration Candidates

> Research/background copied from the Labs vault on 2026-06-06 as starting context for Work Machine. Preserve this as research; repo issues own implementation tasks.

This note captures a first due-diligence pass on products, services, and codebases near the tracker-agnostic workflow-engine vision.

The target architecture under discussion is a workflow system with:

- concise declarative workflow definitions;
- deterministic state-machine behavior;
- tracker adapters such as GitHub Issues/Projects, Trello, Linear, or Hermes Kanban;
- executor adapters for agents, scripts, humans, webhooks, queues, and subworkflows;
- artifact storage adapters for local filesystem, S3, Cloudflare R2, Frame.io, and similar large-file surfaces;
- human and agent review gates;
- notifications through Discord/email/tracker-native channels;
- local and cloud worker support for heavy jobs such as video rendering.

## Executive summary

No single surveyed product cleanly replaces the desired architecture. The market splits into several adjacent categories:

1. low-code automation and integration platforms;
2. durable workflow/state-machine runtimes;
3. AI/agent workflow frameworks;
4. background job and queue systems;
5. media artifact and review platforms.

The likely architecture is not to pick one monolith. It is to compose a tracker-agnostic workflow coordinator with proven substrates and adapters.

Most important candidates to study seriously:

- **Temporal** — durable deterministic state-machine substrate.
- **LangGraph** — LLM-native graph/state/checkpoint/human-interrupt model.
- **Mastra** — TypeScript AI app/agent/workflow framework.
- **Windmill** — open-source scripts + workflows + workers + internal tools reference.
- **Hatchet / Inngest / Trigger.dev** — modern durable job/workflow runners.
- **n8n / Pipedream / Make / Zapier** — integration and workflow UX references.
- **Frame.io / Filestage / ReviewStudio** — review/artifact surfaces, especially for video.
- **S3 / Cloudflare R2** — likely canonical large artifact storage backends.

## Category 1: low-code automation and integration platforms

These are closest to “connect tools and run steps.” They are useful references for integration UX, credentials, triggers, and actions, but they are generally not ideal as the core deterministic state-machine runtime.

Candidates:

- n8n;
- Zapier;
- Make;
- Pipedream;
- Dify;
- Flowise.

### n8n

n8n is an open/source-available low-code automation platform with many integrations, triggers, actions, credentials, and AI-oriented nodes.

Strengths:

- integration catalog;
- credentials and connection management;
- triggers/actions;
- webhook automations;
- visual flow editing;
- self-hosting;
- lots of SaaS plumbing.

Limitations for this project:

- workflows can be tedious to author manually;
- dataflow can feel awkward;
- not ideal as a deterministic state-machine language;
- artifact storage is not first-class;
- human gates exist through waits/forms/webhooks but are not the core abstraction;
- not naturally “tracker-card as system-of-record.”

Verdict:

> Useful as an integration/adapter UX reference, not the core engine.

### Pipedream

Pipedream is a developer-oriented integration platform for event-driven workflows with code steps.

Strengths:

- code steps;
- webhooks;
- event sources;
- scripts;
- API integration;
- logs;
- quick glue automation.

Limitations:

- not a durable human-in-the-loop engine;
- long-running review gates are not central;
- artifact lifecycle is external;
- tracker-centered run state would be custom.

Verdict:

> Good inspiration for developer-friendly integrations and code steps. Not enough as the core state machine.

### Zapier and Make

Zapier and Make are useful for product and UX references:

- integration marketplace;
- no-code workflow editing;
- trigger/action language;
- branching/filter UX;
- error surfaces;
- per-step run history.

But they are proprietary, not deterministic workflow engines, and not a good foundation for custom agentic workflow execution.

Verdict:

> Product/UX references only.

### Dify and Flowise

Dify and Flowise are AI workflow/app builders rather than durable workflow engines.

Useful for:

- visual workflow concepts;
- node palettes;
- prompt/tool composition;
- logs/runs/datasets;
- AI app deployment references.

Limitations:

- not ideal as embedded programmable workflow-engine foundations;
- human gates, artifact storage, and tracker adapters would still need custom design;
- visual-builder-first ergonomics may not match concise workflow.yaml authoring.

Verdict:

> Useful for AI workflow UI references, less likely as the core runtime.

## Category 2: durable workflow/state-machine runtimes

These are closest to the “real engine” part.

Candidates:

- Temporal;
- Netflix/Orkes Conductor;
- Kestra;
- Argo Workflows;
- Airflow;
- Prefect;
- Dagster.

### Temporal

Temporal is the most important candidate to study. It is a durable execution platform for long-running workflows. Workflows are written in code and replayed deterministically; side effects happen through activities.

Mapping to the workflow-engine vision:

| Workflow-engine concept | Temporal concept |
|---|---|
| Workflow run | Workflow execution |
| Step executor | Activity |
| Human gate | Wait for signal |
| Tracker webhook/comment | Signal into workflow |
| Retry/timeout | Built-in activity/workflow policy |
| Subworkflow | Child workflow |
| Long-running run | Durable workflow history |
| External side effect | Activity, not workflow logic |

Strengths:

- long-running workflows;
- deterministic replay;
- retries;
- timers;
- human gates via signals;
- child workflows;
- durable event history;
- local/cloud workers;
- fault tolerance.

Limitations:

- no friendly end-user tracker UI;
- workflows are code-first, not YAML-first;
- determinism constraints are real;
- artifact storage is external;
- not an integration marketplace;
- does not solve GitHub/Trello/Frame.io UX by itself.

Verdict:

> Best candidate for underlying durable execution semantics. Possibly not the immediate Labs prototype substrate, but the gold standard to compare against.

Open question:

> Should `workflow.yaml` eventually compile into or drive a generic Temporal workflow interpreter?

### Netflix/Orkes Conductor

Conductor is an explicit service-orchestration/state-machine platform.

Strengths:

- distributed workflows;
- task queues;
- workers;
- retries;
- branching;
- human-task-like states;
- enterprise visibility.

Limitations:

- heavier operational footprint;
- less AI-native;
- artifact/review UX still external;
- likely overkill for a Labs prototype.

Verdict:

> Strong conceptual competitor to Temporal, but probably too heavy unless this becomes a serious distributed platform.

### Kestra

Kestra is a declarative workflow orchestration platform with UI, logs, schedules, triggers, and plugins.

Strengths:

- YAML/declarative workflows;
- platform workflow UI;
- scripts, containers, and API calls;
- scheduling;
- self-hosting;
- operational visibility.

Limitations:

- more data/platform workflow than agentic human-review loop;
- artifact review/storage remains external;
- tracker adapters would be custom;
- human gates are possible but not central.

Verdict:

> Worth studying if a declarative workflow control plane becomes attractive. Less ideal as the custom agent/human state-machine core.

### Argo Workflows

Argo is strong if everything is Kubernetes/container-native.

Strengths:

- DAGs;
- parallel jobs;
- retries;
- containerized execution;
- S3-style artifacts;
- suspend/resume approval patterns;
- GPU/cloud batch workloads.

Limitations:

- Kubernetes complexity;
- not friendly for arbitrary local Mac mini workers unless wrapped;
- not agent/tracker-centric;
- review UI is not media/human-friendly.

Verdict:

> Strong if rendering becomes containerized/Kubernetes-first. Probably not the first Labs path.

### Prefect, Dagster, and Airflow

These are more data/job orchestration than human/agent workflow coordination.

- **Prefect**: great Python task orchestration, workers, retries, logs.
- **Dagster**: excellent asset lineage/materialization concepts.
- **Airflow**: mature scheduled DAG engine and operational reference.

Verdict:

> Study for task state, logs, retries, artifacts, lineage, and scheduler/executor separation. Do not make them the main mental model for human-in-the-loop agent workflows.

## Category 3: AI and agent workflow frameworks

These are closest to the agent step/orchestration piece.

Candidates:

- Mastra;
- LangGraph;
- CrewAI;
- AutoGen;
- OpenAI Agents SDK;
- PydanticAI;
- LlamaIndex Workflows;
- Haystack.

### Mastra

Mastra describes itself as a modern TypeScript framework for AI-powered applications and agents. The public site emphasizes agents, tools, evals, observability, deployment, framework integration, open-source Apache 2.0 licensing, and productionizing/testing agents.

Strengths:

- TypeScript-native AI app development;
- agents/tools/workflows/memory-style abstractions;
- observability/evals/tracing;
- deployment as app infrastructure;
- likely better DX than stitching LangChain pieces manually;
- good fit if agent workflows live in a modern TS app stack.

Unknowns and evaluation questions:

- How deterministic are workflows?
- Are human gates first-class or app-managed?
- Can workflows pause/resume across days?
- How durable is state?
- What are retry/idempotency semantics?
- How does it handle long-running script/render steps?
- Does it have serious artifact abstractions, or only agent outputs?
- Can it be used without buying into the whole platform?

Verdict:

> Very worth evaluating. Probably not a replacement for the full tracker/artifact/executor architecture, but may be a good agent/workflow framework inside it, especially if the engine goes TypeScript-first.

Likely fit:

```text
agent executor adapter
+ typed tools
+ evals/observability
+ deployable agent services
```

not necessarily:

```text
full durable tracker-agnostic workflow state machine
```

### LangGraph

LangGraph is probably the most directly relevant AI framework. It is a graph/state orchestration framework for LLM applications.

Strengths:

- graph/state-machine workflows;
- explicit state;
- conditional edges;
- loops;
- checkpoints;
- interrupts;
- human-in-the-loop;
- resumability;
- agent/tool nodes.

Limitations:

- not a tracker system;
- artifact storage is external;
- still largely an app framework, not a full product workflow surface;
- Python/LangChain ecosystem dependency may or may not appeal;
- not necessarily the best for deterministic non-agent scripts/render queues.

Verdict:

> Best conceptual benchmark for LLM-native graph workflows. Study deeply. Even if not adopted, its state/checkpoint/interrupt model is directly relevant.

### CrewAI and AutoGen

CrewAI and AutoGen are stronger for multi-agent collaboration than deterministic workflow state machines.

Useful for:

- multi-agent roles;
- autonomous collaboration;
- task delegation;
- research/coding crews;
- conversational agent loops.

Limitations:

- weaker deterministic state-machine semantics;
- human gates are not the main model;
- artifact lifecycle is app-managed;
- external tracker synchronization is custom;
- durable process execution is not the central strength.

Verdict:

> Useful inspiration for multi-agent roles, not the workflow engine core.

### OpenAI Agents SDK and PydanticAI

These are better as agent step implementation layers.

OpenAI Agents SDK is useful for:

- tool calling;
- handoffs;
- tracing;
- guardrails;
- model/provider integration.

PydanticAI is useful for:

- typed agent outputs;
- validation;
- dependency injection;
- structured results.

Verdict:

> Good inside an `agent` executor adapter. Not a full workflow system.

### LlamaIndex Workflows and Haystack

These are useful when workflows involve document-heavy research, intake, retrieval, or RAG.

Useful for:

- RAG;
- document processing;
- retrieval pipelines;
- knowledge artifacts;
- async event-driven steps.

Verdict:

> Useful for research/document/artifact workflows, not the whole system.

## Category 4: durable background job and app workflow systems

These may be especially relevant because of rendering and local/cloud workers.

Candidates:

- Hatchet;
- Inngest;
- Trigger.dev;
- Windmill.

### Hatchet

Hatchet is a strong candidate for the execution layer.

Strengths:

- durable task queues;
- workers;
- retries;
- scheduled/background jobs;
- self-hosting;
- local/cloud worker placement;
- concurrency limits;
- hybrid execution.

Why it matters:

Rendering workflows need workers that may run on:

- the Mac mini;
- a GPU box;
- a cloud VM;
- a container worker;
- a local machine with specific media tooling.

Hatchet-style task queues are a natural fit for that.

Limitations:

- not a tracker UI;
- not a media review platform;
- artifact storage remains external;
- human gates need to be modeled through events/tasks.

Verdict:

> One of the strongest execution-layer candidates if local/cloud worker routing matters.

### Inngest

Inngest is strong for event-driven durable functions.

Strengths:

- app events;
- step functions;
- waits;
- retries;
- concurrency;
- serverless/app integration;
- human-in-the-loop via wait-for-event patterns.

Good mental model:

```text
render.requested
render.completed
review.created
review.approved
changes.requested
publish.requested
```

Limitations:

- not a media review UI;
- large artifacts are external;
- local render-farm style workers may require extra design.

Verdict:

> Very good if the system becomes event-driven product workflow orchestration. Maybe less ideal than Hatchet if local render workers are central.

### Trigger.dev

Trigger.dev is a TypeScript background job framework.

Strengths:

- TypeScript developer experience;
- background jobs;
- queues;
- scheduled tasks;
- webhooks;
- app integration.

Limitations:

- may be less flexible for hybrid local/cloud render worker topology;
- artifact/review/tracker layers remain external.

Verdict:

> Strong if the whole thing becomes a TypeScript app and fast DX matters.

### Windmill

Windmill is a very relevant product to inspect more deeply.

Strengths:

- scripts in multiple languages;
- workflows/flows;
- workers;
- queues;
- self-hosting;
- internal tools;
- resources/secrets;
- schedules/webhooks;
- operational UI.

Limitations:

- not as deterministic as Temporal;
- not tracker-first;
- artifact storage still needs design;
- human gates may be doable but are not as central as the desired architecture.

Verdict:

> Very good reference for script execution, worker architecture, internal UI, and secrets/resources. Possible partial substrate, but likely not the whole vision.

## Category 5: media artifact and review platforms

For video, artifact handling is central rather than secondary.

Candidates:

- S3;
- Cloudflare R2;
- Frame.io;
- Filestage;
- ReviewStudio;
- Google Drive / Dropbox / Box;
- self-hosted object storage / MinIO.

### S3 and Cloudflare R2

S3/R2 are likely the best generic large-artifact storage backends.

Strengths:

- large files;
- signed URLs;
- lifecycle policies;
- CDN/public delivery;
- cheap storage;
- API access;
- separation of artifact payloads from workflow state.

Artifact metadata shape:

```yaml
draft_video:
  uri: r2://bucket/workflow-runs/labs-video/run-123/draft.mp4
  review_url: https://signed-url...
  sha256: ...
  size_bytes: ...
  mime: video/mp4
  produced_by: render_draft
```

Large videos should not live in GitHub or workflow payloads.

Verdict:

> R2/S3 should likely be the canonical artifact store for serious video workflows.

### Frame.io

Frame.io is the key video review candidate.

Strengths:

- video review;
- timecoded comments;
- annotations;
- versions;
- approvals;
- share links;
- stakeholder collaboration;
- media-specific UX.

Limitations:

- not a workflow engine;
- not a general tracker;
- storage/cost/API constraints need evaluation;
- may be best as review surface, not canonical storage.

Verdict:

> Very strong candidate for a `video_review` artifact/review adapter. Pairing R2 as canonical storage with Frame.io as review presentation may be the right split.

### Filestage and ReviewStudio

These are good for broader creative approval workflows across videos, PDFs, images, and documents.

Verdict:

> Worth comparing if workflows involve more than video, or if formal client/stakeholder approval UX matters.

## Candidate matrix

| Candidate | Category | Best use | Main weakness | Fit |
|---|---|---|---|---|
| Temporal | Durable workflow runtime | Long-running deterministic state machine | No friendly tracker/artifact UI | Very high as core substrate |
| LangGraph | AI graph framework | Agent state/checkpoint/human interrupt | Not tracker/artifact platform | Very high as AI workflow benchmark |
| Mastra | TS AI app framework | Agents/tools/workflows/evals/deployment | Need to verify durable HITL semantics | High, especially TS |
| Windmill | Script/workflow/internal tools | Scripts, workers, self-hosted ops | Not tracker-first/deterministic core | High reference |
| Hatchet | Task queue/workflow | Local/cloud workers, durable jobs | No media/tracker UI | High execution layer |
| Inngest | Event workflow | Durable app events, wait-for-event | Artifact/review external | High app workflow layer |
| Trigger.dev | TS jobs | Background jobs in TS apps | May be less hybrid-worker oriented | Medium-high |
| n8n | Automation builder | Integrations, triggers/actions | Tedious manual flows, awkward dataflow | Medium reference |
| Pipedream | Dev automation | Code steps, webhooks, integrations | Not durable HITL engine | Medium reference |
| Kestra | Declarative platform | YAML workflows, UI, scripts | Platform-y, not agent/tracker-first | Medium |
| Argo | K8s workflow | Container/GPU/batch workflows | K8s complexity | Medium if K8s-first |
| Prefect | Python orchestration | Tasks, workers, observability | Data/job oriented | Medium reference |
| Dagster | Asset orchestration | Lineage/materialization metadata | Data asset oriented | Medium reference |
| Frame.io | Media review | Video review/approval | Not execution engine | Very high review layer |
| R2/S3 | Artifact storage | Large durable blobs | Not review UI | Very high storage layer |

## Architectural paths

### Path A: custom lightweight coordinator first

Build a small coordinator around `workflow.yaml`, `run.yaml`, tracker adapters, executor adapters, and artifact adapters.

Pros:

- maximum fit to the desired mental model;
- fastest to prototype inside the existing repo;
- avoids premature platform commitment;
- preserves current workflow-engine work.

Cons:

- eventually rediscovers hard runtime problems:
  - durable waits;
  - retries;
  - idempotency;
  - event history;
  - versioning;
  - concurrent workers;
  - failure recovery.

Best if:

> The immediate goal is Labs momentum and learning the shape before choosing infrastructure.

### Path B: Temporal-backed engine

Use `workflow.yaml` as a declarative layer, but compile or interpret runs using Temporal.

Pros:

- best correctness story;
- human gates map cleanly to signals;
- workers can be local/cloud;
- strong retry/failure semantics;
- subworkflows are real.

Cons:

- more infrastructure and conceptual overhead;
- deterministic replay constraints;
- less lightweight for creative experimentation;
- tracker/artifact/review UX still needs to be built.

Best if:

> The project becomes a serious durable workflow platform rather than a Labs prototype.

### Path C: LangGraph/Mastra as the AI workflow core

Use LangGraph or Mastra for orchestration of agentic steps, plus external tracker/artifact adapters.

Pros:

- AI-native;
- good graph/agent concepts;
- Mastra especially if TypeScript-first;
- LangGraph especially if state/checkpoints/human-in-the-loop matter.

Cons:

- may still not solve deterministic script/render jobs cleanly;
- tracker/artifact system remains custom;
- durable long-running process semantics need scrutiny.

Best if:

> The workflow is primarily agentic reasoning with some tools, not heavy durable job orchestration.

### Path D: Hatchet/Inngest/Trigger.dev execution layer

Use one of these for background work and waits, while the workflow engine owns definitions and tracker/artifact abstractions.

Pros:

- strong job/workflow DX;
- good local/cloud worker story, especially Hatchet;
- better than hand-rolled queues;
- natural for rendering.

Cons:

- need to fit the declarative state machine to their model;
- human review and tracker state remain custom;
- artifact storage remains custom.

Best if:

> The immediate pain is reliable execution of scripts/render jobs across machines.

For artifact-heavy video, **Hatchet + R2/S3 + Frame.io** is a plausible stack.

## Preliminary recommendation

Due diligence should continue in this order:

1. **Temporal** — correctness baseline.
2. **LangGraph** — LLM state-machine baseline.
3. **Mastra** — TypeScript agent app baseline.
4. **Hatchet** — local/cloud worker queue baseline.
5. **Windmill** — scripts/workflows/internal tools baseline.
6. **Frame.io** — video review baseline.

Do not spend too much time on Zapier/Make/Airflow as likely answers. They are useful references, but unlikely to be the foundation.

## First likely serious prototype

A reasonable next prototype:

```text
workflow-engine coordinator
+ GitHub Issues tracker adapter
+ local script executor
+ Hermes/Mastra/LangGraph agent executor spike
+ local filesystem artifact adapter
+ R2 artifact adapter spike
+ Frame.io review adapter spike
+ Discord gate notification
```

Then separately test whether the coordinator should be backed by Temporal, Hatchet, or Inngest.

## Build-vs-buy insight

Existing tools can solve pieces:

- n8n solves integrations;
- Temporal solves durable workflows;
- LangGraph solves AI graph control;
- Mastra solves TypeScript agent app scaffolding;
- Hatchet solves workers/jobs;
- Frame.io solves video review;
- R2/S3 solves large storage.

But the desired product is the composition:

> A tracker-accessible, artifact-heavy, human-gated, agent-and-script workflow system with swappable execution/storage/tracker surfaces.

No obvious off-the-shelf product gives exactly that. The workflow-engine project still makes sense, but the serious version should be positioned as an adapter/coordinator layer over proven substrates, not a from-scratch reinvention of every workflow/runtime/storage/review primitive.

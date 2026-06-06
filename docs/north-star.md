# Work Machine North Star

Work Machine should make complex human/agent/script work feel trackable, reviewable, and shippable without turning the tracker into the execution engine.

## Direction

A user should be able to describe a workflow, start a run, watch it progress through an issue/card surface, review artifacts, approve or request changes, and trust that machine-readable coordinator state remains canonical underneath the human conversation.

## Product promise

- Humans get a familiar tracker surface.
- Agents get repo-local domain docs, specs, and explicit gates.
- Scripts/workers get deterministic run state and artifact contracts.
- Artifacts get indexed with paths, hashes, and review links.
- The system stays small enough to understand before adopting heavier runtimes.

## First proof

One tiny tracker-backed run: local workflow definition, local coordinator state, GitHub issue projection, local script execution, artifact index, human approval gate, and completion.

# Agentic Workflow Canvas (n8n-style) — Design

**Date:** 2026-11-12  
**Status:** Approved  
**Product home:** Inside T3 Code (new UI surface)  
**Delivery:** Big-bang v1 (all listed node types production-ready before ship)

## Problem

T3 Code already runs agents, delegates, human pauses, PR linking, and browser tooling through threads and chat. Users who want **repeatable multi-step agent pipelines** (plan → ask human → fan-out workers → QA with evidence → PR) have no first-class graph editor or durable run cockpit. Existing “workflows” in the product mean coordinator/subagent runs inside a thread, not an n8n-like node canvas.

## Goals

- Ship an **n8n-like node canvas** inside T3 for **agent orchestration**.
- Persist workflows as **project-level entities**; runs create/bind threads underneath.
- Live run experience on the canvas (nodes light up; side panel shows turn/gate/evidence).
- Support the **hero pipeline** end-to-end (see below).
- Leave room for classic automation nodes (HTTP, Slack, cron) **later** without redesigning the engine.

## Non-goals (v1)

- Classic automation builder / integration marketplace.
- Mobile canvas editing (mobile: list runs + resolve human gates only).
- Rewriting an in-flight run when the definition is edited.
- Silent model fallback when a configured provider/model is unavailable.
- Claiming QA passed when browser/evidence tools were unavailable.
- Mid-run plan rewrite without cancel + new run.

## Hero workflow (acceptance story)

Task-delegate pipeline:

1. Human starts a run with a task brief (attachments optional).
2. **Brainstorm** agent checks the task and produces a plan + `needsContext`.
3. **Branch:** if more context needed → **HumanGate** → **loop back to Brainstorm**; else continue.
4. **Parallel** fan-out of worker **Agent** steps on cheaper models; join when all succeed.
5. **QA** agents produce evidence; if web → real browser evidence; else skip browser path via Branch (never fake evidence).
6. **PR** agent opens a PR that includes/links the evidence.

Loop safety: Brainstorm ↔ HumanGate edges have a configurable max-iteration cap.

## Approach

**Server-owned Workflow engine.** The graph is data. `WorkflowEngine` advances a `WorkflowRun` from committed events and emits effects that call existing T3 services (threads, turns, providers, pending human requests, PR tools, browser/QA). Clients edit definitions and subscribe to run projections; they do not conduct the graph.

Rejected alternatives:

- Client-side conductor (breaks remote/multi-device and gate durability).
- Meta-agent that “follows” a graph as prose (too unreliable for branches/loops/evidence).

## Architecture

```
Web/Desktop canvas + inspector
Mobile: runs + gate resolve
        ↓ typed WS / contracts
Workflow + WorkflowRun schemas, RPCs, projections
        ↓ commands → events
WorkflowEngine (pure decide; effects only)
        ↓
Existing runtime: threads/turns/providers, delegates,
checkpoints, PR linking, browser tools, pending requests
```

### Invariants

- A **Workflow** is a project-owned definition (nodes, edges, layout, version).
- A **WorkflowRun** pins a **definition snapshot** at start; definition edits do not mutate in-flight runs.
- Progress of a run is authoritative on the run projection, not on chat UI state.
- Node execution may spawn or bind threads/turns; those are linked from node attempts.
- Automation node types are out of v1 but the engine/schema should not hard-block a later type registry.

## Domain model

### Workflow

- `workflowId`, `projectId`, `title`, `createdAt`, `updatedAt`, `version`
- `graph`: nodes[], edges[], viewport/layout metadata
- Validation on save and on `StartRun`

### WorkflowNode (v1 types)

| Type        | Purpose              | Key config                                                                                             |
| ----------- | -------------------- | ------------------------------------------------------------------------------------------------------ |
| `Start`     | Run input            | Task brief + attachment refs seed context                                                              |
| `Agent`     | Provider turn        | Provider, model, prompt template (from context), worktree strategy, fresh thread vs delegate under run |
| `HumanGate` | Pause for human      | Prompt/fields schema, timeout policy, resolve actions                                                  |
| `Branch`    | Conditional edge     | Expression / structured field (e.g. `needsContext`), loop cap when edge returns to prior node          |
| `Parallel`  | Fan-out + join       | Child agent node refs; v1 join = all must succeed                                                      |
| `Wait`      | External/manual wait | PR checks, schedule, manual resume                                                                     |
| `Transform` | Context mapping      | Map/reshape fields between steps / into prompts                                                        |

### WorkflowEdge

- `from`, `to`, optional `label` / branch key (`then` / `else` / named)

### WorkflowRun

- `runId`, `workflowId`, pinned `definitionSnapshot`, `status`
- Shared `context` bag (JSON-compatible, typed paths for known outputs)
- Per-node **attempts** (re-entrancy on loop creates a new attempt; history preserved; outputs versioned)
- Linked threads/turns per attempt
- Artifacts (QA evidence refs, screenshots, logs) with size caps / summaries
- Active gates, failure reason, timestamps

### Run context

- Start seeds `context` from the task brief and attachments.
- Finished nodes write outputs into context (plan, `needsContext`, human answers, worker results, evidence index, PR URL).
- Agent prompts are templates over context; Transform reshapes before later nodes.

## Server components

### WorkflowService

CRUD workflows; `startRun` / `cancelRun`; `resolveGate`; list/subscribe workflows and runs. Thin transport over commands; permissions via existing client-guarded RPC scopes (operate).

### WorkflowEngine

Pure decisioning on graph + run state (same spirit as Orchestrator): given a command or completion event, emit run events + outbox effects. No I/O in the decide step.

### Node adapters (effect handlers)

Per-type handlers that perform I/O through existing services:

- Agent → create/bind thread, send turn with configured provider/model
- HumanGate → pending request visible on web/desktop/mobile
- Wait → subscribe/wait for PR checks or timer/manual
- Branch / Transform / Parallel join → engine-local with context writes
- Artifacts → store refs for inspector + PR step consumption

## Client surfaces

### Web + Desktop (v1)

- Workflow list under a project (alongside threads)
- Canvas editor (React Flow–style): add/configure nodes, wire edges, validate
- Run cockpit: same canvas with live node statuses; click node → inspector (config in edit mode; turn/gate/evidence in run mode)
- Entry: project navigation + command palette where other project artifacts live

### Mobile (v1)

- List workflows/runs; open run status; approve/reject/answer HumanGates
- No canvas editor

## Data flow (hero path)

1. `StartRun` → pin snapshot, seed context, activate Start → Brainstorm
2. Brainstorm Agent completes → write plan / `needsContext` → Branch
3. If needs context → HumanGate → on resolve, edge **back to Brainstorm** (new attempt); else → Parallel workers
4. Parallel join when all workers succeed → QA Branch
5. QA produces artifacts; Transform maps evidence into PR prompt context
6. PR Agent completes → PR URL in context → run succeeds

Clients subscribe to the run projection. Mobile `resolveGate` uses the same RPC; engine resumes.

## Error handling

### Per-node policies

- **fail-run** (default for Agent / QA / PR): mark node failed → run failed → cancel in-flight Parallel siblings
- **retry**: N attempts with backoff; then fall through to node policy
- **gate**: open HumanGate with error (retry / skip / abort)
- **skip-branch**: optional paths (e.g. non-web browser QA)

### Parallel failure (locked)

One worker fails → **cancel siblings + fail the run**. “Continue with partials” is not v1 default (optional later).

### Must-handle edge cases

- Server restart mid-run: resume from last committed node state; in-flight turns use existing orchestration recovery; open gates stay open
- Provider/model unavailable: clear failure → retry or gate; **no silent model fallback**
- Client auth/pairing lost: run continues on server; any paired device can resolve gates
- Environment offline (remote): `waiting_environment`; UI shows blocked reason; resume on reconnect
- Graph validation: missing Branch edges, Parallel without join, illegal cycles (except capped loop-backs), Parallel worktree collisions
- Invalid Branch structured output (e.g. missing `needsContext`): HumanGate (clarify / force continue / abort), not silent continue
- Loop re-entrancy: new node-attempt; context accumulates; prior outputs versioned
- QA browser unavailable: skip-branch or gate; never fake evidence; PR must not claim tests passed
- PR exists / push rejected: gate with retry / alternate / abort; avoid duplicate PRs
- Double Run: idempotency / UI disable; separate runs only when explicitly started
- Permission denied (write/PR): gate with grant path
- HumanGate timeout: wait forever (default), remind, or fail-run
- Cancel run: interrupt turns/delegates, close gates, mark cancelled (idempotent)
- Concurrent gate resolve: first write wins; second gets conflict

### Should-handle

- Evidence size caps / thumbnails / compact PR evidence index
- Gate field validation (non-empty / schema) at RPC before resume

### Explicitly unsupported in v1

- User manually editing the repo while a run is live (document; agent checkpoints still apply per turn)
- Mid-flight plan change without cancel + rerun
- Cost runaway controls beyond basic validation (optional max concurrency can be added later)

## Testing

- **WorkflowEngine:** table-driven advances for the hero graph; failure policies; loop cap; cancel idempotency; snapshot pin; recovery / `waiting_environment`
- **Contracts:** graph validate; gate resolve validation and conflict
- **Web:** `*.logic.test.ts` for projection → status mapping and command wiring; sparse browser tests for add node / connect edge / open inspector
- **Integrated proof:** one real-client hero pass via `test-t3-app` after implementation
- Engine tests use Replay/dev provider fixtures; do not require paid multi-provider CI runs

## Extensibility (post-v1)

- Automation nodes (triggers, HTTP, Slack, cron) register as additional node types + adapters
- Parallel `continue-with-partials` policy
- Mobile canvas editing
- Richer cost/budget caps

## Success criteria

v1 is done when a user can, in web/desktop:

1. Create and save the hero graph under a project
2. Start a run with a task brief
3. Watch nodes advance live on the canvas
4. Answer a context HumanGate (including from mobile) and see Brainstorm re-run
5. See parallel workers complete, QA evidence appear, and a PR open with evidence linked
6. Observe correct failure behavior when a parallel worker fails (siblings cancelled, run failed)

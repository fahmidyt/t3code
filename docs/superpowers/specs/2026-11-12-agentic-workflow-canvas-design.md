# Agentic Workflow Canvas (n8n-style) — Design

**Date:** 2026-11-12  
**Status:** Approved  
**Product home:** Inside T3 Code (new UI surface)  
**Delivery:** Big-bang product ship (all v1 node types ready before user-facing release). Implementation work is still sequenced by subsystem in the plan (contracts → engine → adapters → web → mobile gates).

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
- Cron / webhook / HTTP automation triggers (including Wait-on-schedule).
- Mobile canvas editing (mobile: list runs + resolve human gates only).
- Rewriting an in-flight run when the definition is edited.
- Silent model fallback when a configured provider/model is unavailable.
- Claiming QA passed when browser/evidence tools were unavailable.
- Mid-run plan rewrite without cancel + new run.
- Special node types named Brainstorm / QA / PR — those are configured **Agent** nodes (+ Branch / Transform as needed).
- MCP exposure of workflow RPCs in v1 (client WS RPC only; MCP can follow later).

## Hero workflow (acceptance story)

Task-delegate pipeline (all steps use the v1 node types below):

1. Human starts a run with a task brief (attachments optional) via **Start**.
2. **Agent** (“Brainstorm”) checks the task and writes `plan` + boolean `needsContext` into context via a declared structured output mapping.
3. **Branch** on `brainstorm.needsContext`: if true → **HumanGate** → edge **back to Brainstorm Agent** (new attempt); else continue.
4. **Parallel** fan-out along outbound edges to worker **Agent** nodes (cheaper models); join when all succeed.
5. **Branch** web vs not: web path runs **Agent** (“QA”) with browser tools (`onError: "skip"` if browser unavailable); non-web takes else and skips that Agent. Evidence artifacts update `context.artifacts.evidence` via the engine. **Transform** maps `artifacts.evidence` into the PR prompt context.
6. **Agent** (“PR”) opens a PR that includes/links the evidence index.

**Loop safety:** On a Branch that can return to an earlier node, `maxLoopIterations` defaults to **3** (configurable on that Branch, integer ≥ 1). Exceeding the cap → fail-run with reason `loop_cap_exceeded` (optional final HumanGate can be wired explicitly if desired).

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
Workflow schemas, RPCs, run projections
        ↓ commands → events (+ outbox effects)
WorkflowEngine (pure decide)
        ↓ effect worker / node adapters
Existing runtime: threads/turns/providers, delegates,
checkpoints, PR linking, browser tools, pending requests
```

### Invariants

- A **Workflow** is a project-owned definition (nodes, edges, layout, version).
- A **WorkflowRun** pins a **definition snapshot** at start; definition edits do not mutate in-flight runs.
- Progress of a run is authoritative on the run projection, not on chat UI state.
- Node execution may spawn or bind threads/turns; those are linked from node attempts.
- Exactly **one** `Start` node per graph.
- Node-type registry is extensible later; v1 ships a fixed set.

### Persistence

- Store workflow definitions and run projections in the environment userdata SQLite (`statev2` family), same home rules as other project artifacts — **not** the developer’s live `~/.t3` from a worktree by accident.
- **Event-sourced runs:** append-only `WorkflowRunEvent` log per run; projections (status, context, attempts, artifacts, active gates) rebuilt/updated in the same commit transaction as events + outbox effects (mirrors orchestration sink pattern).
- Definition rows: current graph + monotonic `version`. Saving bumps `version`. Concurrent save: optimistic concurrency on `version`; loser retries/rebases in UI.
- Snapshot at `StartRun`: deep-copy graph JSON into the run row (size soft-cap; reject start if snapshot exceeds cap — default 1 MiB encoded).
- Context bag soft-cap (default 512 KiB encoded); Transform/Agent writes that would exceed → node failure with `context_too_large`.
- Artifacts: metadata + paths/URLs in DB; large blobs on disk under userdata with per-run directory.
- **Evidence context contract:** whenever an `ArtifactAdded` event is committed, the engine patches `context.artifacts.items` (array of `{ id, kind, title, hrefOrPath, producedByAttempt }`) and rebuilds `context.artifacts.evidence` as items whose `kind` is in the evidence set. Hero Transform reads `from: "artifacts.evidence"` — adapters do not write that path themselves.
- **Artifact `kind` enum (v1):** `screenshot` | `browser_trace` | `log` | `file` | `other`. Evidence set for `context.artifacts.evidence` = `screenshot` | `browser_trace` | `file` (QA attachments). `log` and `other` stay in `items` only.

### Subscription model

- `workflows.subscribe` / `workflowRuns.subscribe` (project-scoped) push projection updates over existing WS subscription patterns (same family as scheduled tasks / orchestration projections).

## Domain model

### Workflow

- `workflowId`, `projectId`, `title`, `createdAt`, `updatedAt`, `version`
- `graph`: `{ nodes, edges, layout }`
- Validated on save and on `StartRun`

### Graph validation (reject save / start)

- Exactly one `Start` with exactly one outbound edge (unlabeled) to the first real node
- No dangling edges; all endpoint ids exist
- `Branch`: exactly two outbound edges with keys `then` and `else` only
- `Parallel`: ≥ 1 outbound edge from the Parallel **fork** handle to `Agent` nodes; each of those Agents has exactly one success outbound edge that returns to this Parallel’s **join** handle (Transform-only nodes may sit on that return path; no other Agent may); the Parallel node also has exactly one outbound edge from handle **`next`** taken only after join succeeds
- Cycles: a back-edge (edge whose target can reach its source via existing directed edges) is legal only if it is the `then` or `else` edge of a `Branch` that sets `maxLoopIterations`, or it is reachable only through such a Branch edge; otherwise reject
- Unreachable nodes → warning on save, **error** on StartRun
- Parallel worktree collision: two worker Agents under the same Parallel with identical resolved worktree identity → error (see worktree strategies)

### WorkflowNode (v1 types)

Hero labels like Brainstorm / QA / PR are **titles** on `Agent` nodes, not separate types.

| Type        | Purpose               | Key config                                                                                                      |
| ----------- | --------------------- | --------------------------------------------------------------------------------------------------------------- |
| `Start`     | Run input             | Seeds `context.task` (string) + `context.attachments` (ids)                                                     |
| `Agent`     | Provider turn         | See Agent config below                                                                                          |
| `HumanGate` | Pause for human       | Node config + resolve matrix below                                                                              |
| `Branch`    | Conditional edge      | See Branch below                                                                                                |
| `Parallel`  | Fan-out + join        | fork → Agents → join; then `next` edge; one worker failure cancels siblings + fail-run                          |
| `Wait`      | Block until condition | **v1:** `manual` resume **or** `pullRequestChecks` (thread/PR already linked in context). **No schedule/cron.** |
| `Transform` | Context mapping       | See Transform below                                                                                             |

Each node also carries:

```
onError: "fail-run" | "retry" | "gate" | "skip"
retry?: { maxAttempts: number; backoffMs: number }  // required when onError is "retry"
retryExhausted?: "fail-run" | "gate"               // required when onError is "retry"; default "fail-run"
```

Defaults: `Agent` / critical path → `fail-run`.

**`onError` semantics:**

- `fail-run` — fail attempt → fail run; if the attempt is a Parallel fork worker, cancel sibling fork attempts first
- `retry` — retry up to `maxAttempts` with `backoffMs`; when exhausted, apply `retryExhausted` (`fail-run` or open HumanGate `error`)
- `gate` — open HumanGate `error` (actions depend on whether the node is a Parallel fork worker — see HumanGate matrix)
- `skip` — mark attempt `skipped` and continue along the node’s **primary outbound success edge** as if succeeded, writing `context.skips[<nodeId>] = true`. Not allowed on `Start`, `Parallel`, or any Agent that is a Parallel fork worker (validation error). Use `skip` for optional linear steps (e.g. browser QA Agent when tools unavailable via adapter mapping to skip).

**Parallel node `onError`:** ignored / invalid on save — worker failure always cancels siblings + fail-run. Fork worker Agents may use `fail-run`, `retry` (with `retryExhausted: "fail-run" | "gate"`), or `gate`, but never `skip`.

### Agent config

- `provider`, `model` (required; unavailable → fail per policy, no silent fallback)
- `promptTemplate`: string with `{{path.to.field}}` substitutions from context (missing path → empty string; use Transform to require fields earlier)
- `structuredOutput?: { fields: Record<string, "string" | "boolean" | "number">; writeTo: string }`  
  Example Brainstorm: fields `{ needsContext: "boolean", plan: "string" }`, `writeTo: "brainstorm"`  
  On success, **merge** (shallow) the parsed object into `context[writeTo]` (create object if missing); do not delete sibling keys under that path. Top-level `writeTo: ""` merges into context root.
- `threadMode`: `"newThread"` | `"delegateUnderRun"` (run has a coordinator thread id)
- `worktree`:
  - `"project"` — project root (default)
  - `"new"` — fresh worktree per attempt
  - `"key:<string>"` — shared named worktree among nodes with the same key  
    Collision rule: under one Parallel, two children may not both use `"project"` or the same `key:` string.

### Branch

- `predicate`: **only** path predicates, not a general language:
  - `{ type: "pathTruthy", path: "brainstorm.needsContext" }`
  - `{ type: "pathEquals", path: "task.kind", value: "web" }`
  - `{ type: "pathExists", path: "pr.url" }`
- Paths are dot-paths into the run `context` object.
- Outbound edges: `then` (predicate true), `else` (false).
- If the path is missing / wrong type for the predicate → **do not take else silently**; open HumanGate kind `branch_repair` with only `force_then` | `force_else` | `abort` (no “send back to previous Agent” action in v1).
- `maxLoopIterations` when any outbound path can reach a prior node (default 3).
- **Loop counter:** stored on the run as `context._meta.loopCounts[<branchNodeId>]` (integer, default 0). Each time that Branch is entered and evaluates to take an edge that can reach a prior node on the graph, increment before taking the edge; if `count > maxLoopIterations`, emit fail-run `loop_cap_exceeded` instead. Counters persist in `ContextPatched` events so restart restores them.

### Transform

- Ordered list of mapping ops (no scripts in v1):
  - `{ op: "set", to: "pr.evidenceIndex", from: "artifacts.evidence" }`
  - `{ op: "setLiteral", to: "qa.mode", value: "browser" }`
  - `{ op: "pick", to: "summary", from: "brainstorm", keys: ["plan"] }`
- Missing `from` → node fails with `transform_missing_path` unless `optional: true` on that op.
- No loops/conditionals inside Transform (use Branch nodes).

### HumanGate node config

```
title: string
bodyTemplate: string          // {{context.path}} substitution
fields: Array<{
  id: string
  label: string
  type: "string" | "text" | "boolean"
  required: boolean
  writeTo: string             // dot-path under context where submit writes the value
}>
timeout: { mode: "forever" } | { mode: "fail-run"; timeoutMs: number }
```

System-opened gates (`error`, `branch_repair`, `structured_output_repair`, `wait_manual`) use engine-generated title/body/fields; `wait_manual` has zero fields.

### HumanGate (unified resolve model)

Gate payload always includes: `gateId`, `runId`, `nodeId`, `attempt`, `kind`, `title`, `body`, `fields[]`, `actions[]`.

**Kinds → allowed actions (closed matrix):**

| Kind                       | When                                                                       | Actions                             | Effect                                                                                                                                         |
| -------------------------- | -------------------------------------------------------------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `input`                    | HumanGate node on hero/context path                                        | `submit`, `abort`                   | `submit` merges each field’s value into `context` at that field’s `writeTo` path and continues success edge; `abort` → fail run `gate_aborted` |
| `error`                    | `onError: "gate"` / `retryExhausted: "gate"` on a **non-fork-worker** node | `retry`, `skip`, `abort`            | `retry` restarts the failed attempt; `skip` applies node `onError: "skip"` semantics; `abort` → fail run                                       |
| `error` (fork worker)      | same, but node is a Parallel fork worker Agent                             | `retry`, `abort` only               | No `skip` (would break Parallel join). `abort` or exhausted failure cancels siblings + fail-run                                                |
| `branch_repair`            | Branch predicate path missing/wrong type                                   | `force_then`, `force_else`, `abort` | Take the chosen Branch edge without re-evaluating; `abort` → fail run                                                                          |
| `structured_output_repair` | Agent structured output missing/invalid                                    | `retry`, `abort`                    | `retry` re-runs the Agent attempt; `abort` → fail run. No force-continue of bad output in v1                                                   |
| `wait_manual`              | Wait node `mode: "manual"`                                                 | `submit`, `abort`                   | `submit` completes the Wait attempt successfully; `abort` → fail run                                                                           |

No other actions exist in v1. UI only shows the actions listed for the gate’s kind.

**Timeout:** uses the HumanGate node’s `timeout` config for kind `input` (`forever` or `fail-run` → emit `GateTimedOut` then fail run). System gates default to `forever`. No `remind` mode in v1.

`resolveGate` RPC body: `{ gateId, action, fieldValues? }`. Action must be in the kind’s set; empty required fields on `submit` → RPC validation error (engine not resumed). Concurrent resolve → conflict.

**DeleteWorkflow / edit while running:** `DeleteWorkflow` rejected if any non-terminal run exists for that workflow. Editing the definition is allowed (bumps `version`) and does not affect in-flight runs.

### Parallel (graph shape)

- Parallel is a **canvas node** with:
  - `fork` handle: outbound edges to worker `Agent` nodes
  - `join` handle: each worker’s success path must edge back to this Parallel’s join (directly or via Transform-only nodes; no other Agent between worker and join in v1)
- No alternate `joinNodeId` shape in v1.
- **Parallel attempt lifecycle:** `AttemptStarted` on the Parallel node when fork fires; worker Agents get their own attempts; when all workers succeed and join is satisfied, `AttemptSucceeded` on the Parallel node, then the engine follows the Parallel **`next`** outbound edge. One worker failure → cancel sibling workers + `AttemptFailed` on Parallel + fail-run (no `next`).
- Workers are normal graph nodes (not nested JSON-only children)

### Start / coordinator lifecycle

- `StartRun` creates the run, pins the snapshot, seeds context from input, and creates attempt 0 on the `Start` node which immediately succeeds (no provider call).
- Optionally creates a **coordinator thread** (`coordinatorThreadId`) for the run when any Agent uses `threadMode: "delegateUnderRun"`; otherwise threads are created per Agent attempt as needed.
- After Start succeeds, the engine follows the single outbound edge to the next node (Brainstorm Agent in the hero graph).

### Wait (narrowed)

- `mode: "manual"` → opens `wait_manual` HumanGate
- `mode: "pullRequestChecks"` → config `{ prUrlPath: string }` (dot-path to PR URL or number already in context, e.g. `pr.url`); uses existing PR check watching; completes when required checks green (or fails per `onError`)
- No cron/schedule mode in v1

### WorkflowEdge

- `id`, `from`, `to`, `fromHandle?`, `toHandle?`, `key?` (`then` | `else` | unlabeled)
- Parallel canonical handles: `fromHandle`/`toHandle` values `fork` | `join` | `next` as described under Parallel
- `StartRun` `idempotencyKey` is scoped per `workflowId` (same key on another workflow is unrelated)

### WorkflowRun

- `runId`, `workflowId`, `definitionSnapshot`, `definitionVersion`, `status`
- `context` (JSON object)
- `attempts[]`: `{ nodeId, attempt, status, threadId?, turnIds?, outputs?, error?, startedAt, finishedAt? }`
- `artifacts[]`: `{ id, kind, title, hrefOrPath, thumbHref?, producedByAttempt, bytes? }`
- `activeGateId?`
- `failureReason?`
- Timestamps; linked `coordinatorThreadId?`

### Statuses

- Run: `pending` | `running` | `waiting_gate` | `waiting_environment` | `succeeded` | `failed` | `cancelled`
- Attempt: `pending` | `running` | `succeeded` | `failed` | `cancelled` | `skipped`

## Commands, events, effects

### Commands (WorkflowService → engine)

- `UpsertWorkflow` / `DeleteWorkflow`
- `StartRun` `{ workflowId, input, idempotencyKey }`
- `CancelRun` `{ runId }`
- `ResolveGate` `{ gateId, action, fieldValues? }`
- `ResumeAfterEnvironment` (internal, when host reconnects)
- Completions from adapters: `ReportAgentAttemptFinished` / `Failed`, `ReportWaitFinished` / `Failed`, `ReportArtifact`

### Events (persisted)

- `WorkflowDefined` / `WorkflowDeleted`
- `RunStarted` (with snapshot)
- `AttemptStarted` / `AttemptSucceeded` / `AttemptFailed` / `AttemptCancelled` / `AttemptSkipped`
- `ContextPatched`
- `ArtifactAdded`
- `GateOpened` / `GateResolved` / `GateTimedOut`
- `RunWaitingEnvironment` / `RunResumed`
- `RunSucceeded` / `RunFailed` / `RunCancelled`

### Effects (outbox → adapters)

- `StartAgentAttempt`
- `CancelAgentAttempt`
- `OpenHumanGate`
- `CloseHumanGate`
- `StartWaitPullRequestChecks`
- `CancelWait`
- `PersistArtifact`

## Server components

### WorkflowService

Public API + permission mapping. Methods (WS):

| Method                                                              | Scope                           |
| ------------------------------------------------------------------- | ------------------------------- |
| `workflows.list` / `workflows.get` / `workflows.subscribe`          | read (paired session)           |
| `workflows.upsert` / `workflows.delete`                             | `AuthOrchestrationOperateScope` |
| `workflowRuns.list` / `workflowRuns.get` / `workflowRuns.subscribe` | read                            |
| `workflowRuns.start` / `workflowRuns.cancel`                        | `AuthOrchestrationOperateScope` |
| `workflowRuns.resolveGate`                                          | `AuthOrchestrationOperateScope` |

Add each mutating method to `CLIENT_GUARDED_RPC_SCOPES`.  
v1: **not** registered on orchestrator MCP.

### WorkflowEngine

Pure function/module: `(state, command|completion) → { events, effects }`. Wired like Orchestrator: serialize per `runId`, commit events + projection + outbox, then effect worker runs adapters and feeds completions back as commands.

### Node adapters

I/O only for effects above. **Artifacts are not a node type**; Agent tool results (screenshots, logs) call `ReportArtifact` / `PersistArtifact`. Transform/Branch/Parallel join are engine-local (no adapter).

## Client surfaces

### Web + Desktop (v1)

- Workflow list under a project (alongside threads)
- Canvas editor (React Flow–style / `@xyflow`): add/configure nodes, wire edges, validate
- Run cockpit: same canvas with live statuses; click node → inspector (edit config vs attempt/gate/evidence)
- Command palette entry for “Workflows” / “New workflow”

### Mobile (v1)

- List workflows/runs; run status; resolve HumanGates with the unified action enum
- No canvas editor

## Data flow (hero path)

1. `StartRun` → pin snapshot, seed context, activate Start → Brainstorm Agent
2. Brainstorm succeeds with structured output → Branch on `needsContext`
3. true → HumanGate `input` → on `submit`, edge back to Brainstorm (new attempt, loop counter++); false → Parallel workers
4. Parallel join → Branch web/not → QA Agent and/or skip → artifacts → Transform evidence index
5. PR Agent → `context.pr.url` → `RunSucceeded`

## Error handling

### Per-node `onError`

- **fail-run / retry / gate / skip:** see Domain model `onError` semantics (single source of truth)

### Parallel failure (locked)

One worker fails → cancel siblings + fail the run.

### Must-handle edge cases

- Server restart mid-run: resume from last committed events; in-flight turns use existing orchestration recovery; open gates stay open
- Provider/model unavailable: clear failure; no silent fallback
- Client auth lost: run continues; any paired operate-scoped session can `resolveGate`
- Environment offline: `waiting_environment`; resume on reconnect
- Invalid structured output: HumanGate `structured_output_repair` (`retry`/`abort`); invalid Branch predicate path: `branch_repair`
- Loop re-entrancy: new attempt; context accumulates; prior outputs versioned under attempts
- QA browser unavailable: node `onError: "skip"` or `"gate"`; never fake evidence; PR prompt must only cite `context.artifacts.evidence`
- PR exists / push rejected: Agent failure → node `onError` (recommend `gate`)
- Double Run: `idempotencyKey` on start — if a non-terminal run with the same key exists, return that run (no second run); UI also disables Start while a non-terminal run is selected/active
- Permission denied: prefer `onError: gate`
- Cancel: idempotent interrupt + close gates
- Concurrent gate resolve: first wins; second conflict

### Should-handle

- Evidence size caps / thumbnails / compact evidence index
- Gate field schema validation at RPC

### Unsupported in v1

- Manual repo edits during a live run (document)
- Mid-flight plan rewrite without cancel + new run
- Parallel continue-with-partials
- Run cost budgets (optional max concurrency may be added later)

## Testing

- **WorkflowEngine:** table-driven hero graph; Branch true/false; loop cap; Parallel fail cancels siblings; Transform missing path; Wait manual + PR checks; cancel idempotency; snapshot pin; `waiting_environment`
- **Contracts:** graph validate cases; resolveGate validation/conflict; optimistic workflow version
- **Web:** `*.logic.test.ts` for projection mapping + commands; sparse browser tests for add node / connect edge / open inspector
- **Integrated proof:** one `test-t3-app` hero pass after implementation
- Engine tests use Replay/dev provider fixtures

## Extensibility (post-v1)

- Automation nodes (triggers, HTTP, Slack, cron) + Wait schedule mode
- Parallel `continue-with-partials`
- Mobile canvas editing
- MCP tools for workflows
- Richer Transform/Branch languages if path predicates prove insufficient

## Success criteria

v1 is done when a user can, in web/desktop:

1. Create and save the hero graph under a project
2. Start a run with a task brief
3. Watch nodes advance live on the canvas
4. Answer a context HumanGate (including from mobile) and see Brainstorm re-run
5. See parallel workers complete, QA evidence appear, and a PR open with evidence linked
6. Observe correct failure when a parallel worker fails (siblings cancelled, run failed)

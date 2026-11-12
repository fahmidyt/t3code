# Agentic Workflow Canvas Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship an n8n-style project-level Workflow canvas in T3 Code with a server-owned engine that runs the hero task-delegate pipeline (brainstorm → context gate loop → parallel workers → QA evidence → PR).

**Architecture:** Event-sourced `Workflow` / `WorkflowRun` in userdata SQLite; pure `WorkflowEngine` emits events + outbox effects; adapters call existing threads/providers/PR/pending-request APIs; web/desktop canvas + run cockpit; mobile list workflows/runs + resolve gates. Spec: [docs/superpowers/specs/2026-11-12-agentic-workflow-canvas-design.md](../specs/2026-11-12-agentic-workflow-canvas-design.md).

**Tech Stack:** Effect Schema contracts, Effect services on `apps/server`, WS RPC, `packages/client-runtime` atoms/commands, React + `@xyflow/react` in `apps/web`, React Native gate/run UI in `apps/mobile`.

**Ship gate:** Big-bang product release. Implement Chunks 1→8 in order.

**Naming:** Canvas RPCs are `workflows.*` / `workflowRuns.*` only. Do not collide with orchestration `stopWorkflow` / thread “workflow” subagent concepts.

---

## File structure (locked)

| Path                                                                                | Responsibility                                                                      |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `packages/contracts/src/workflowIds.ts`                                             | Brands: WorkflowId, WorkflowRunId, node/edge/gate/artifact ids                      |
| `packages/contracts/src/workflowGraph.ts`                                           | Node configs, edges, graph, Workflow definition row shape                           |
| `packages/contracts/src/workflowRun.ts`                                             | Run projection, attempts, artifacts, gates, commands/events/effects tags + payloads |
| `packages/contracts/src/workflowValidation.ts`                                      | Pure `validateWorkflowGraph`                                                        |
| `packages/contracts/src/workflowValidation.test.ts`                                 | Validation tables                                                                   |
| `packages/contracts/src/workflowGraph.test.ts`                                      | Schema round-trips for every node type                                              |
| `packages/contracts/src/workflowRun.test.ts`                                        | Run/gate/artifact schema round-trips                                                |
| `packages/contracts/src/rpc.ts`                                                     | WS methods + Rpc.make                                                               |
| `packages/contracts/src/clientRpcPermissions.ts`                                    | Operate scopes for mutating methods                                                 |
| `packages/client-runtime/src/state/commandPermissions.test.ts`                      | Assert new tags guarded (extend)                                                    |
| `apps/server/src/workflows/WorkflowDefinitionStore.ts`                              | Definition CRUD + optimistic version                                                |
| `apps/server/src/workflows/WorkflowRunStore.ts`                                     | Events, projection, outbox, artifacts metadata                                      |
| `apps/server/src/workflows/WorkflowEngine.ts`                                       | Pure decide                                                                         |
| `apps/server/src/workflows/BranchLogic.ts`                                          | Predicates + loop counts                                                            |
| `apps/server/src/workflows/TransformLogic.ts`                                       | Mapping ops                                                                         |
| `apps/server/src/workflows/ParallelLogic.ts`                                        | Fork/join bookkeeping                                                               |
| `packages/contracts/src/workflowHeroGraph.ts`                                       | Single hero graph factory (fixture + web template + server tests)                   |
| `apps/server/src/workflows/WorkflowService.ts`                                      | Public API                                                                          |
| `apps/server/src/workflows/WorkflowEffectWorker.ts`                                 | Drain outbox (pattern: `apps/server/src/orchestration-v2/EffectWorker.ts`)          |
| `apps/server/src/workflows/adapters/*.ts`                                           | Agent, Gate, Wait, Artifact                                                         |
| `apps/server/src/ws.ts` / `server.ts`                                               | Wire RPCs + layers                                                                  |
| `packages/client-runtime/src/state/workflows.ts`                                    | Atoms/commands/selectors                                                            |
| `apps/web/src/routes/_chat.workflows.$environmentId.$projectId.tsx`                 | List                                                                                |
| `apps/web/src/routes/_chat.workflows.$environmentId.$projectId.$workflowId.tsx`     | Editor + cockpit                                                                    |
| `apps/web/src/components/workflows/*`                                               | Canvas UI                                                                           |
| `apps/web/src/components/CommandPalette.tsx`                                        | Add “Workflows” / “New workflow” commands                                           |
| `apps/mobile/src/features/workflows/*`                                              | List workflows/runs, run status, gate resolve                                       |
| `apps/mobile/src/features/layout/createWorkspaceStackNavigator.tsx` (+ ios/android) | Register workflow screens                                                           |

---

## Chunk 1: Contracts + graph validation

### Task 1: Ids + graph schemas

**Files:** Create `workflowIds.ts`, `workflowGraph.ts`, `workflowGraph.test.ts`; export from `index.ts`

- [ ] **Step 1: Write failing round-trip tests** covering Start, Agent (structuredOutput + worktree + onError retry), HumanGate fields/timeout, Branch predicates, Parallel (empty config), Wait manual + pullRequestChecks, Transform ops, edge handles `fork`|`join`|`next`, keys `then`|`else`

- [ ] **Step 2: Run** `vp test run packages/contracts/src/workflowGraph.test.ts` — expect FAIL

- [ ] **Step 3: Implement schemas** matching the spec Domain model (split from run types)

Agent `onError` schema:

```ts
export const NodeOnError = Schema.Union([
  Schema.Struct({ onError: Schema.Literal("fail-run") }),
  Schema.Struct({
    onError: Schema.Literal("retry"),
    retry: Schema.Struct({
      maxAttempts: Schema.Int.check(Schema.isGreaterThan(0)), // total attempts including first
      backoffMs: Schema.Int.check(Schema.isGreaterThanOrEqualTo(0)),
    }),
    retryExhausted: Schema.Literal(["fail-run", "gate"]),
  }),
  Schema.Struct({ onError: Schema.Literal("gate") }),
  Schema.Struct({ onError: Schema.Literal("skip") }),
]);
```

- [ ] **Step 4: Run** — expect PASS

- [ ] **Step 5: Commit** `feat(contracts): add workflow graph schemas`

### Task 2: Run / gate / event schemas

**Files:** Create `workflowRun.ts`, `workflowRun.test.ts`

- [ ] **Step 1: Failing tests** for `StartRunInput`, attempt statuses, artifact kinds, gate kinds/actions matrix payloads, command/event/effect tag literals from spec sections “Commands, events, effects”

Literal unions must include at least:

```ts
// commands
"UpsertWorkflow" |
  "DeleteWorkflow" |
  "StartRun" |
  "CancelRun" |
  "ResolveGate" |
  "ResumeAfterEnvironment" |
  "ReportAgentAttemptFinished" |
  "ReportAgentAttemptFailed" |
  "ReportWaitFinished" |
  "ReportWaitFailed" |
  "ReportArtifact" |
  "GateTimedOut";

// effects
"StartAgentAttempt" |
  "CancelAgentAttempt" |
  "OpenHumanGate" |
  "CloseHumanGate" |
  "StartWaitPullRequestChecks" |
  "CancelWait" |
  "PersistArtifact";
```

- [ ] **Step 2–4: Implement, PASS, commit** `feat(contracts): add workflow run and engine message schemas`

### Task 3: Graph validation

**Files:** `workflowValidation.ts`, `workflowValidation.test.ts`

- [ ] **Step 1: Table cases** (save vs start):

| Case                                                                      | mode | expect         |
| ------------------------------------------------------------------------- | ---- | -------------- |
| zero/two Starts                                                           | both | error          |
| Start without exactly one outbound                                        | both | error          |
| dangling edge                                                             | both | error          |
| Branch missing then/else                                                  | both | error          |
| Parallel missing fork/join/next shape                                     | both | error          |
| Parallel worktree collision                                               | both | error          |
| illegal cycle (back-edge not via Branch then/else with maxLoopIterations) | both | error          |
| legal loop: Branch then → HumanGate → Brainstorm (maxLoopIterations set)  | both | ok             |
| `onError: skip` on Start/Parallel/fork worker                             | both | error          |
| Parallel node carries onError                                             | both | error          |
| `onError: retry` missing retry/retryExhausted                             | both | error          |
| unreachable node                                                          | save | warning; start | error |

Return type:

```ts
export type WorkflowValidationIssue = {
  readonly severity: "error" | "warning";
  readonly code: string;
  readonly message: string;
  readonly nodeId?: string;
};
export function validateWorkflowGraph(
  graph: WorkflowGraph,
  mode: "save" | "start",
): ReadonlyArray<WorkflowValidationIssue>;
```

- [ ] **Step 2: Run FAIL → implement → PASS**

- [ ] **Step 3: Commit** `feat(contracts): validate workflow graphs`

### Task 4: WS RPCs + permissions

**Files:** `rpc.ts`, `clientRpcPermissions.ts`, `packages/client-runtime/src/state/commandPermissions.test.ts`

Payload shapes (decode/encode via Schema):

| Method                                    | Payload essentials                                     |
| ----------------------------------------- | ------------------------------------------------------ |
| `workflows.upsert`                        | `{ workflowId?, projectId, title, graph, version? }`   |
| `workflows.delete`                        | `{ workflowId }`                                       |
| `workflows.list` / `get`                  | `{ projectId }` / `{ workflowId }`                     |
| `workflows.subscribe`                     | `{ projectId }`                                        |
| `workflowRuns.start`                      | `{ workflowId, input: StartRunInput, idempotencyKey }` |
| `workflowRuns.cancel`                     | `{ runId }`                                            |
| `workflowRuns.resolveGate`                | `{ gateId, action, fieldValues? }`                     |
| `workflowRuns.list` / `get` / `subscribe` | by `workflowId` or `projectId`                         |

- [ ] **Step 1: Add WS_METHODS + Rpc.make**

- [ ] **Step 2: Guard upsert/delete/start/cancel/resolveGate with `AuthOrchestrationOperateScope`**

- [ ] **Step 3: Extend `commandPermissions.test.ts`** so each new mutating tag resolves operate scope

- [ ] **Step 4: Run** `vp test run packages/client-runtime/src/state/commandPermissions.test.ts packages/contracts/src/workflowValidation.test.ts`

- [ ] **Step 5: Commit** `feat(contracts): workflow WS RPCs and operate scopes`

---

## Chunk 2: Persistence + WorkflowService shell

**Persistence references:** mirror transactional commit style of `apps/server/src/orchestration-v2/EventSink.ts` and userdata access used by `apps/server/src/scheduledTasks/ScheduledTaskService.ts`. **v1 outbox:** dedicated `workflow_outbox` table (do not share orchestration outbox tables).

### Task 5: Definition + run stores

**Files:** `WorkflowDefinitionStore.ts`, `WorkflowRunStore.ts`, tests beside each

DDL (conceptual):

```sql
workflows(id TEXT PK, project_id TEXT, title TEXT, graph_json TEXT, version INT, created_at INT, updated_at INT);
workflow_runs(id TEXT PK, workflow_id TEXT, definition_version INT, snapshot_json TEXT, status TEXT, context_json TEXT, coordinator_thread_id TEXT NULL, failure_reason TEXT NULL, idempotency_key TEXT, created_at INT, updated_at INT);
UNIQUE(workflow_id, idempotency_key) WHERE idempotency_key IS NOT NULL;
workflow_run_events(run_id TEXT, seq INT, type TEXT, payload_json TEXT, at INT, PRIMARY KEY(run_id, seq));
workflow_outbox(id TEXT PK, run_id TEXT, type TEXT, payload_json TEXT, created_at INT, processed_at INT NULL);
workflow_artifacts(id TEXT PK, run_id TEXT, kind TEXT, title TEXT, path TEXT, produced_by_attempt INT, bytes INT, created_at INT);
```

Artifact blobs: `<userdata>/workflows/<runId>/<artifactId>` on disk; DB holds path.

- [ ] **Step 1: Failing tests** — optimistic version conflict; append event + update projection + insert outbox in one transaction; delete workflow rejected if non-terminal run; snapshot > 1 MiB rejected on start; terminal run frees idempotency key for reuse (delete unique row or allow new start with same key after terminal)

- [ ] **Step 2: Implement stores**

- [ ] **Step 3: Run** `vp test run apps/server/src/workflows/WorkflowDefinitionStore.test.ts apps/server/src/workflows/WorkflowRunStore.test.ts`

- [ ] **Step 4: Commit** `feat(server): workflow definition and run persistence`

### Task 6: WorkflowService shell (engine-backed Start only)

**Files:** `WorkflowService.ts`, `WorkflowService.test.ts`

Until Chunk 3 lands, call a **fixed stub decide** used only in Task 6 tests:

```ts
// stub: on StartRun → events [RunStarted, AttemptStarted(Start), AttemptSucceeded(Start), ContextPatched]
// + effects [] and activate first edge target by emitting AttemptStarted for that node if Agent → StartAgentAttempt
```

Replace stub with real `WorkflowEngine.decide` in Chunk 3 without changing service API.

- [ ] **Step 1: Tests** upsert/get/list/delete; startRun validates `mode:"start"`; seeds `context.task` + attachments; idempotency returns existing non-terminal; delete blocked

- [ ] **Step 2: Implement with `Effect.fn("WorkflowService.*")`**

- [ ] **Step 3: Run** `vp test run apps/server/src/workflows/WorkflowService.test.ts` — PASS

- [ ] **Step 4: Commit** `feat(server): WorkflowService CRUD and startRun`

---

## Chunk 3: WorkflowEngine

### Task 7: Hero happy path + Wait

**Files:** `WorkflowEngine.ts`, `BranchLogic.ts`, `TransformLogic.ts`, `ParallelLogic.ts`, `packages/contracts/src/workflowHeroGraph.ts`, `WorkflowEngine.test.ts`

Import hero graph from `packages/contracts/src/workflowHeroGraph.ts` only (add in Task 7 if not present).

- [ ] **Step 1: Failing table tests**

1. Start → Brainstorm Agent effect
2. Structured output merge into `brainstorm.*` → Branch false → Parallel fork `StartAgentAttempt` × N
3. All workers succeed → Parallel `AttemptSucceeded` → `next`
4. `ReportArtifact` rebuilds `context.artifacts.items` + `evidence` (kinds `screenshot`|`browser_trace`|`file` only)
5. Transform → PR Agent → `RunSucceeded`
6. Wait `manual` → `OpenHumanGate` kind `wait_manual`; `ResolveGate submit` → Wait succeeded
7. Wait `pullRequestChecks` → `StartWaitPullRequestChecks`; `ReportWaitFinished` → continue

- [ ] **Step 2: Implement pure `decide(state, input)`**

- [ ] **Step 3: Run** `vp test run apps/server/src/workflows/WorkflowEngine.test.ts` — PASS for happy/Wait cases

- [ ] **Step 4: Commit** `feat(server): WorkflowEngine hero path and Wait`

### Task 8: Failures, gates, loops, environment

- [ ] **Step 1: Add failing cases**

- Branch true → input gate → submit → loop; `context._meta.loopCounts[branchId]`; cap → `loop_cap_exceeded`
- Invalid structured output → `structured_output_repair`
- Invalid branch path → `branch_repair`
- Parallel worker fail → `CancelAgentAttempt` siblings + fail run
- Fork worker `error` gate actions omit `skip`
- Non-fork `onError: skip` continues success edge + `context.skips[nodeId]`
- `onError: retry` then `retryExhausted`
- `GateTimedOut` on input gate with fail-run timeout → fail run
- `CancelRun` idempotent + `CloseHumanGate` / `CancelWait` / `CancelAgentAttempt` effects as needed
- Enter `waiting_environment`: when any in-flight Agent/Wait adapter reports environment offline (completion command `ReportEnvironmentUnavailable`), engine emits `RunWaitingEnvironment`, leaves attempts paused (no cancel), holds unprocessed outbox; UI status `waiting_environment`
- `ResumeAfterEnvironment` → `RunResumed`, re-drive paused attempts / remaining outbox (no duplicate StartAgentAttempt if already running remotely)
- `transform_missing_path`; `context_too_large`
- Snapshot pin ignores later definition edits

Add to `workflowRun.ts` command tag: `ReportEnvironmentUnavailable`.

- [ ] **Step 2: Implement → PASS → commit** `feat(server): WorkflowEngine failure and gate paths`

---

## Chunk 4: Adapters + worker + WS

### Task 9: Adapters

**Files:** `adapters/AgentAdapter.ts`, `GateAdapter.ts`, `WaitAdapter.ts`, `ArtifactAdapter.ts` + tests

- [ ] **Step 1: Failing tests**

- Agent: render `promptTemplate`; honor `threadMode` (`newThread` vs `delegateUnderRun` using `coordinatorThreadId`); honor `worktree` `project`|`new`|`key:`; cancel → interrupt; on host offline emit `ReportEnvironmentUnavailable`
- Service/start: if snapshot has any Agent with `delegateUnderRun`, create `coordinatorThreadId` on `StartRun` before first Agent effect (spec Start lifecycle)
- Gate: open/close; schedule fail-run timeout → `GateTimedOut` command into engine
- Wait: manual gate; PR checks via existing watch; `CancelWait`
- Artifact: write blob under userdata path; `ReportArtifact`

- [ ] **Step 2: Implement → PASS → commit** `feat(server): workflow adapters`

### Task 10: Effect worker + WS wire-up

**Files:** `WorkflowEffectWorker.ts`, modify `WorkflowService.ts`, `server.ts`, `ws.ts`

- [ ] **Step 1: Integration test** with mocked adapters: start hero → drain → hit gate or terminal

- [ ] **Step 2: Implement worker** copying serialize/drain ideas from `orchestration-v2/EffectWorker.ts`

- [ ] **Step 3: Thin WS handlers** for all Chunk 1 methods

- [ ] **Step 4: Run service/ws tests → commit** `feat(server): workflow effect worker and WS RPCs`

---

## Chunk 5: client-runtime

### Task 11: Workflow atoms/commands

**Files:** `packages/client-runtime/src/state/workflows.ts`, wire from `server.ts` if required by existing patterns, `workflows.test.ts`

Expose:

- subscriptions: `workflows.subscribe`, `workflowRuns.subscribe`
- reads: list/get helpers
- commands: `upsertWorkflow`, `deleteWorkflow`, `startWorkflowRun`, `cancelWorkflowRun`, `resolveWorkflowGate` via `createEnvironmentRpcCommand` + `permissionAtom`

- [ ] **Step 1: Failing tests** including **cancel** + permission atoms for all mutating commands

- [ ] **Step 2: Implement → PASS** `vp test run packages/client-runtime/src/state/workflows.test.ts`

- [ ] **Step 3: Commit** `feat(client-runtime): workflow subscriptions and commands`

---

## Chunk 6: Web canvas + cockpit

### Task 12: Dependency

- [ ] Add `@xyflow/react` to `apps/web` via workspace package manager; commit lockfile — `chore(web): add @xyflow/react`

### Task 13: List + navigation

**Files:**

- `apps/web/src/routes/_chat.workflows.$environmentId.$projectId.tsx`
- `apps/web/src/components/workflows/WorkflowList.tsx`
- `apps/web/src/components/workflows/workflowList.logic.ts`
- `apps/web/src/components/workflows/workflowList.logic.test.ts`
- Modify: `apps/web/src/components/CommandPalette.tsx` (add commands “Workflows” / “New workflow in…” next to New thread items)
- Modify: project sidebar entry — locate project row actions in `apps/web/src` sidebar components (same area as project threads entry; if multiple, prefer the component that already links into project-scoped routes) and add “Workflows”

- [ ] Logic tests → implement → `vp test run apps/web/src/components/workflows/workflowList.logic.test.ts` → commit `feat(web): workflow list and navigation`

### Task 14: Canvas editor (all v1 node types)

**Files:** `WorkflowCanvas.tsx`, `workflowNodes.tsx`, `WorkflowInspector.tsx`, `workflowCanvas.logic.ts` + `.logic.test.ts` + `WorkflowCanvas.browser.test.tsx`, workflow detail route

- [ ] **Must support creating/editing:** Start, Agent, HumanGate, Branch, Parallel, Wait, Transform — each with inspector fields from spec configs
- [ ] Logic tests: graph ↔ xyflow; handles fork/join/next; validate before save
- [ ] Browser test: add Agent, connect edge, open inspector (`apps/web/test/render.tsx` disposable)
- [ ] Commit `feat(web): workflow canvas editor for all v1 node types`

### Task 15: Run cockpit

**Files:** `WorkflowRunCockpit.tsx`, `workflowRunCockpit.logic.ts` + test, `WorkflowGatePanel.tsx`, `StartWorkflowRunDialog.tsx` (task brief + optional attachments → `StartRunInput`)

- [ ] Start disabled while non-terminal run active for workflow (spec)
- [ ] Node click shows attempt / gate / **evidence** artifacts
- [ ] Gate panel filters actions by kind matrix
- [ ] cancel + resolveGate wired
- [ ] Commit `feat(web): workflow run cockpit, start dialog, and evidence inspector`

---

## Chunk 7: Mobile

### Task 16: Workflows + runs + gates

**Files (create):**

- `apps/mobile/src/features/workflows/WorkflowListScreen.tsx`
- `apps/mobile/src/features/workflows/WorkflowRunListScreen.tsx`
- `apps/mobile/src/features/workflows/WorkflowRunStatusScreen.tsx`
- `apps/mobile/src/features/workflows/WorkflowGateScreen.tsx`
- `apps/mobile/src/features/workflows/workflowGateActions.ts`
- `apps/mobile/src/features/workflows/workflowGateActions.test.ts`

**Files (modify):** `apps/mobile/src/features/layout/createWorkspaceStackNavigator.tsx` (+ `.ios.tsx` / `.android.tsx` if stacks split), register routes; add entry from project/thread sidebar patterns (`ThreadNavigationSidebar.tsx` or project switcher — add “Workflows” row).

Mirror pending UI patterns from `PendingUserInputCard.tsx` / `PendingApprovalCard.tsx` for gate field entry.

- [ ] **Step 1:** `workflowGateActions.test.ts` — action sets per gate kind (incl. fork-worker error omits skip)

- [ ] **Step 2:** Screens for **list workflows**, **list runs**, **run status**, **resolve gate**

- [ ] **Step 3:** No canvas

- [ ] **Step 4:** `vp test run apps/mobile/src/features/workflows/workflowGateActions.test.ts` → commit `feat(mobile): workflow runs and human gates`

---

## Chunk 8: Hero proof

### Task 17: Template + proof

- [ ] Web “New from hero template” imports `workflowHeroGraph` from contracts (created in Chunk 3)

- [ ] Engine/service test: full hero with Replay mocks through evidence → PR context
- [ ] One `test-t3-app` pass: open workflows, start run, resolve gate (per `.agents/skills/test-t3-app/SKILL.md`)
- [ ] Optional: one `test-t3-mobile` gate resolve for success criterion 4
- [ ] `vp run check:changed`
- [ ] Commit `feat(workflows): hero template and client proof`

---

## Success criteria mapping

| Spec criterion                  | Owned by                |
| ------------------------------- | ----------------------- |
| Create/save hero graph          | Chunks 1, 6             |
| Start with task brief           | Chunks 2, 6 Task 15     |
| Live canvas statuses            | Chunks 3–6              |
| Mobile gate + brainstorm re-run | Chunks 3, 7             |
| Parallel + QA evidence + PR     | Chunks 3, 4, 6, 8       |
| Parallel fail cancels siblings  | Chunk 3 tests + cockpit |

---

## Notes

- Worktree servers use worktree `.t3`, never live `~/.t3` read-write.
- No MCP workflow tools in v1.
- Desktop inherits web.
- Prefer existing PR-watch and pending-request infra inside adapters.

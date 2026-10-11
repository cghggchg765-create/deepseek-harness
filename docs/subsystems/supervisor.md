# Supervised Terminals

English | [中文](supervisor.zh.md)

The vocabulary shared by the supervised-terminal components' tools, detection rules, and durable records. Installed through the [`@deepseek-ai/dsh-supervisor`](../../packages/supervisor/supervisor/README.md) bundle, which composes a terminal core, planning, orchestration, and operations; the core package owns this vocabulary and its README owns the detection and configuration rules. This page records only the types a consumer of the stored records or of the model-facing results needs.

Source: [`packages/supervisor/supervisor-core/src/types.ts`](../../packages/supervisor/supervisor-core/src/types.ts)

## What one settlement reports

A mark describes what a terminal did at settlement, never whether it is healthy. The supervising model reads marks and decides; nothing in the plugin acts on one.

```ts type-equiv
/**
 * Deterministic observation derived from one terminal settlement. A mark is a
 * fact about what the terminal returned, never a judgement: the hosted model
 * decides whether one is abnormal and what to do about it.
 */
type StateMark =
  /** The terminal settled at an input prompt (`stdin_read` or `inferred_idle`). */
  | 'prompt-idle'
  /** The output matches an interactive confirmation prompt. */
  | 'waiting-input'
  /** The output carries an activity marker of the hosted agent CLI. */
  | 'working'
  /** The wait timed out with no prompt; the child may still be working or stuck. */
  | 'silence'
  /** The top-level session exited with code 0. */
  | 'exit-ok'
  /** The top-level session exited with a non-zero code or a signal. */
  | 'exit-bad'
  /** The heartbeat file for this session is older than the configured timeout. */
  | 'heartbeat-stale'
```

An L1 hit is a rule match with the verbatim fragment that triggered it, so the model can judge the evidence instead of the rule name alone.

```ts type-equiv
/** One deterministic L1 rule match. */
interface L1RuleHit {
  /** Built-in rule id that matched. */
  ruleId: string
  /** Configured severity of that rule. */
  severity: L1Severity
  /** Verbatim output fragment that triggered the match, bounded for display. */
  evidence: string
  /** Epoch milliseconds of the scan that produced this hit. */
  timestamp: number
}
```

## What a plan holds

A plan decomposes one goal into subtasks whose `dependsOn` edges must form a DAG. Acceptance criteria carry the method that verifies them so a resumed run verifies the same way the original run intended.

```ts type-equiv
/** One independently verifiable unit of a decomposed task. */
interface SubTask {
  /** Stable id unique within the plan; referenced by `dependsOn`. */
  id: string
  /** Short imperative title. */
  title: string
  /** What to do, including the acceptance points the hosted agent must satisfy. */
  description: string
  /** Ids of subtasks that must complete before this one starts; forms a DAG. */
  dependsOn: string[]
  /** Skills the hosted agent must use to complete this subtask. */
  requiredSkills: string[]
  /** Agent kind best suited to this subtask, when the planner has a preference. */
  preferredAgent?: AgentKind
  /** Rough effort estimate. */
  estimatedEffort: EstimatedEffort
  /** Subtask-level acceptance criteria. */
  acceptanceCriteria: string[]
  /** Current lifecycle state. */
  status: SubTaskStatus
}
```

## What a checkpoint pins

A checkpoint is the durable resume point: the git commit, the terminal it was driving, and the output log that carried the context. Records are self-describing JSON, so a restore is possible by hand when the plugin is not loaded.

```ts type-equiv
/** Git facts captured with a checkpoint. */
interface GitRef {
  /** Absolute path of the repository the checkpoint belongs to. */
  repoPath: string
  /** Branch the checkpoint commits on. */
  branch: string
  /** Commit the checkpoint pins, or an empty string when no commit was made. */
  commitHash: string
  /** Whether tracked files were dirty when the checkpoint was taken. */
  hasUncommitted: boolean
}
```

```ts type-equiv
/** Terminal facts captured with a checkpoint so a relaunch can reproduce them. */
interface TerminalStateRecord {
  /** Hosted agent kind running in the terminal. */
  agentKind: AgentKind
  /** Exact launch command used last, including substituted model and prompt. */
  lastLaunchCommand: string
  /** Model requested at launch or through a switch command, when known. */
  lastModel?: string
  /** Last observed session status. */
  state: string
  /** Input the supervisor still owed the terminal when the checkpoint was taken. */
  pendingInput?: string
}
```

## What a review round records

A finding states one defect the reviewing model decided on, and a ticket carries the findings through repair. The ticket owns the retry budget, so the escalation point is durable rather than a decision the model has to remember.

```ts type-equiv
/** One review finding. */
interface Finding {
  /** Stable id unique within the ticket. */
  id: string
  /** How badly this blocks delivery. */
  severity: FindingSeverity
  /** Which review dimension produced it. */
  category: FindingCategory
  /** One-line statement of the defect. */
  title: string
  /** Evidence: file, line, command, or output fragment. */
  evidence: string
  /** What is missing or wrong relative to the acceptance criteria. */
  gap: string
  /** Concrete repair the responsible agent should attempt. */
  suggestedFix: string
  /** Acceptance criterion this finding violates, when one applies. */
  acceptanceCriterionId?: string
}
```

```ts type-equiv
/** One review round's findings and their repair state. */
interface IssueTicket {
  /** Identity of this ticket. */
  ticketId: string
  /** Task the findings belong to. */
  taskId: string
  /** Findings still under repair, in declaration order. */
  findings: Finding[]
  /** Instructions handed to the responsible agent for the next attempt. */
  redoInstructions: string
  /** Checks the repair must re-run before the ticket can resolve. */
  regressionScope: string[]
  /** Repairs attempted so far. */
  retryCount: number
  /** Repairs allowed before the ticket escalates. */
  maxRetries: number
  /** Current lifecycle state. */
  status: TicketStatus
  /** Epoch milliseconds of creation. */
  createdAt: number
  /** Epoch milliseconds of the latest transition. */
  updatedAt: number
}
```

## Stored records and their retention

`supervisor_checkpoint` writes one JSON file per checkpoint under `checkpoints/<taskId>/`, and retention keeps `checkpointMaxPerTask` records per task and drops records older than `checkpointMaxAgeDays`. `supervisor_issue` writes one JSON file per ticket under `tickets/<taskId>/`, and `supervisor_skills` writes one inventory per agent kind under `skills/`. The heartbeat record is the liveness file the watchdog compares against `heartbeatTimeoutSec`; it is written per settlement, so its age is the age of the last observed output rather than the age of the process.

## Automatic pacing of the supervising agent

The harness's own model requests fail the way the hosted CLIs' do, and a supervising agent that stalls supervises nothing. `dsh-llm-retry` retries inside the failing step; when it gives up, the turn ends `error`, and the `supervisor-pacing` component owns what happens next: it reads the provider's stated resume instant (or falls back to a bounded jittered backoff), re-arms an active goal the failure disarmed, and wakes the agent for one more turn, inside a per-session budget that ends in a chat alert.

It registers no tool, publishes no service, and adds no session event. The decision reads the `pacing` session projection, which folds every `turn/end` into the last outcome and its failure facts — the sanctioned way to know a session’s last turn after a restart, where synchronous event reads are deprecated. Provider-facing limit wording stays in the core package’s `limit.ts`, so one module owns that vocabulary.

## Automatic shutdown

`supervisor-ops` owns the shutdown gate and `@deepseek-ai/dsh-supervisor-shutdown` owns what acts on it. The gate is the operator's policy: `shutdown.enabled`, `shutdown.command`, and `shutdown.gracePeriodMin` live in the operations configuration, `shutdownDecision` answers one trigger with an allowed-or-refused decision carrying the command, and `supervisor_shutdown` reports that decision without executing anything. The shutdown component consumes the decision through `ctx.supervisorOps` and runs the returned command through `ctx.shell`; it repeats none of the policy and invents no command of its own.

The component arms an attempt along four configured paths. A `scheduleAt` wall clock resolves to its next local occurrence; `scheduleAfterMinutes` or `scheduleAfterSeconds` arms a countdown from plugin load, with the seconds value winning when both are set; `watchPlans` arms when every stored plan's subtasks are `completed` or `skipped` and drops the attempt when work reopens; and `idleTimeoutMin` arms after that many minutes with no running agent. The `supervisor_power` tool arms the same attempts on the supervising model's request — `confirm` for work it asserts is finished, `schedule` for a countdown — and reports or cancels what is armed.

Before an attempt fires, `remindAtMin` (10, 5, and 1 by default) announces each threshold once per attempt into the log and, when `notifyChatId` names a conversation, to chat. A fresh attempt that starts already inside a threshold marks that threshold announced, so a countdown armed inside the ten-minute window never reports "ten minutes" for a deadline that is already closer; re-arming restarts the reminders instead of replaying delivered ones.

`cancelCommand` is the path back. It runs through the same shell seam even when this component armed nothing, because an earlier run may have left the OS with a shutdown timer of its own, and only the operator's command stops a machine already counting down. `supervisor_power cancel` takes that path on request, and reopened work cancels an attempt the plan watcher armed. `dryRun` reports every command the component would run — shutdown and cancellation — through the same announce path instead of executing it.

The safety model has four parts. The component never invents an OS command: the gate returns the operator's command, and a refusal carries no command at all. `enabled` defaults to false, and the gate refuses unless the operator enabled shutdown and configured a non-empty command, so an unconfigured deployment can never power a machine off. The gate is asked when an attempt fires, not when it is armed, so an attempt still inside `gracePeriodMin` is a not-yet rather than a refusal. And one attempt is spent per condition: a fired plan-completion attempt does not re-arm until work reopens, and an idle attempt restarts the idle clock, so no condition runs the command twice.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxsupervisor--supervisorruntime"></a>

### `ctx.supervisor` — `SupervisorRuntime`

The supervised-terminal control plane: the registry of driven terminals, the one detection pipeline every terminal tool funnels its settlement through, the durable heartbeat, output-log, and plan stores, and the watchdog that wakes an idle agent when a terminal goes quiet.

```ts cordis-catalog
/**
 * Drop one terminal from the registry, e.g. after it was closed.
 * @param sessionId - terminal session id to forget.
 */
forget(sessionId: string): void

/**
 * Run one settlement through the detection pipeline and record it.
 * @param owner - agent driving the terminal.
 * @param sessionId - terminal session id.
 * @param text - settlement output.
 * @param waitReason - wait reason reported by the terminal operation.
 * @param status - top-level session status at settlement.
 * @param now - epoch milliseconds of the observation.
 * @returns marks, rule hits, and spill facts for the tool result.
 */
async observe( owner: Agent, sessionId: TerminalSessionId, text: string, waitReason: TerminalWaitReason, status: TerminalSessionStatus, now: number, ): Promise<SettlementReport>

/**
 * Scan retained scrollback without recording a settlement. A read observes
 * text the supervisor already has; it neither appends to the output log nor
 * advances the heartbeat, because doing either would let a poll of a stuck
 * terminal pass for progress.
 * @param owner - agent driving the terminal.
 * @param sessionId - terminal session id.
 * @param text - retained scrollback that was read.
 * @param status - top-level session status at the read.
 * @param now - epoch milliseconds of the read.
 * @returns marks and rule hits for the inspected text.
 */
async inspect( owner: Agent, sessionId: TerminalSessionId, text: string, status: TerminalSessionStatus, now: number, ): Promise<InspectionReport>

/**
 * Record which CLI runs in a terminal, so its output is classified with that
 * agent's characteristics from the next observation on. Declaring it is how a
 * terminal the supervisor did not launch becomes classifiable.
 * @param owner - agent driving the terminal.
 * @param sessionId - terminal session id.
 * @param kind - hosted agent kind.
 * @param status - status observed at the declaration.
 */
declareAgent(owner: Agent, sessionId: TerminalSessionId, kind: AgentKind, status: TerminalSessionStatus): void

/**
 * Register or fetch the registry entry for one terminal.
 * @param owner - agent driving the terminal.
 * @param sessionId - terminal session id.
 * @param status - status observed most recently, used to seed a new entry.
 * @returns the registry entry, created on first use.
 */
track(owner: Agent, sessionId: TerminalSessionId, status: TerminalSessionStatus): SupervisedSession

/**
 * Check one terminal's heartbeat and alert the caller when it went stale.
 * @param sessionId - terminal session id.
 * @param onStale - called with the registry entry when the heartbeat is stale and the cooldown elapsed.
 * @param now - epoch milliseconds of the watchdog pass.
 */
async checkStaleness( sessionId: string, onStale: (session: SupervisedSession) => void, now: number, ): Promise<void>
```

Types: [Agent](core.md) · [TerminalSessionId](terminal.md) · [TerminalSessionStatus](terminal.md) · [TerminalWaitReason](terminal.md)

Source: [`packages/supervisor/supervisor-core/src/index.ts`](../../packages/supervisor/supervisor-core/src/index.ts)

<a id="ctxsupervisorops--supervisoropsservice"></a>

### `ctx.supervisorOps` — `SupervisorOpsService`

The operations service: the usage ledger, the delivery report store, the chat bridge seam, and the four operations tools.

```ts cordis-catalog
/**
 * Register the deployment's chat transport. The plugin owns the default
 * routing: an inbound message reaches the agents that own supervised
 * terminals as a steer, so a human can ask for status or redirect the run
 * from the chat. A deployment that needs its own routing passes a handler and
 * takes over that job. This is the documented extension point a chat
 * connector mounts through.
 * @param transport - chat provider supplied by the deployment.
 * @param handler - optional routing override receiving every parsed message.
 * @returns the disposer that unsubscribes and forgets the transport.
 */
useChatTransport( transport: ChatTransport, handler?: (message: ChatMessage, command: { command: ChatCommand; argument: string }) => void, ): () => void

/**
 * Record one usage observation under the task the supervised terminals are
 * currently on, moving the ledger's task window first so the observation is
 * never dropped by the move.
 * @param entry - usage to record.
 */
recordUsage(entry: UsageEntry): void

/**
 * Current budget state, read from the ledger.
 * @param now - epoch milliseconds the day and the alert thresholds are evaluated against.
 * @returns the task and daily spend with their alert and exceeded flags.
 */
budgetStatus(now: number): BudgetStatus
```

Source: [`packages/supervisor/supervisor-ops/src/index.ts`](../../packages/supervisor/supervisor-ops/src/index.ts)

<a id="ctxsupervisortasks--supervisortaskservice"></a>

### `ctx.supervisorTasks` — `SupervisorTaskService`

Task plans, checkpoints, and issue tickets for supervised runs: the durable state an interrupted or reviewed run resumes from, stored under the control plane's state directory.

Source: [`packages/supervisor/supervisor-tasks/src/index.ts`](../../packages/supervisor/supervisor-tasks/src/index.ts)
<!-- END GENERATED cordis-surface -->

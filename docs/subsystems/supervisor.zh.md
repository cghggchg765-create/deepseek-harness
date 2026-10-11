# 受管终端

[English](supervisor.md) | 中文

受管终端各组件（工具、检测规则与持久记录）共用的词汇。它通过 [`@deepseek-ai/dsh-supervisor`](../../packages/supervisor/supervisor/README.zh.md) bundle 安装，该 bundle 由终端核心、计划、编排与运维四个组件组成；这套词汇由核心包拥有，检测与配置规则见其 README。本页只记录存储记录或模型可见结果的消费方需要知道的类型。

来源：[`packages/supervisor/supervisor-core/src/types.ts`](../../packages/supervisor/supervisor-core/src/types.ts)

## 一次结算报告什么

标记描述的是终端在结算时做了什么，而不是它是否健康。监工模型读取标记并自行判断；插件中没有任何东西会依据标记行动。

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

L1 命中是一次规则匹配加上触发它的原文片段，因此模型可以评判证据本身，而不只是规则名。

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

## 计划里有什么

计划把一个目标拆成子任务，其 `dependsOn` 边必须构成 DAG。验收标准携带验证它们的方法，使续跑时按原意验证。

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

## 断点钉住什么

断点是持久的续跑点：git 提交、它当时驱动的终端，以及承载上下文的输出日志。记录是自描述的 JSON，因此在插件未加载时也能手工恢复。

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

## 一轮审查记录了什么

一条 finding 描述评审模型判定的一处缺陷，ticket 则带着这些 finding 走完修复流程。重试预算由 ticket 持有，因此升级点是持久化的，而不是模型需要记住的决定。

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

## 存储记录与保留策略

`supervisor_checkpoint` 在 `checkpoints/<taskId>/` 下为每个断点写一个 JSON 文件，保留策略按任务保留 `checkpointMaxPerTask` 条记录，并删除超过 `checkpointMaxAgeDays` 的记录。心跳记录是看门狗对照 `heartbeatTimeoutSec` 比较的存活文件；它按每次结算写入，因此它的年龄是最后一次观察到的输出的年龄，而不是进程的年龄。

## 监工 Agent 自身的自动起搏

harness 自身的模型请求会像被托管的 CLI 一样失败，而一个停摆的监工谁也监不了。`dsh-llm-retry` 在失败的步骤内重试；当它放弃时回合以 `error` 结束，接下来由 `supervisor-pacing` 组件接管：它读取 provider 声明的恢复时刻（否则退回有界抖动退避）、重新武装被失败解除的活跃目标（goal），再唤醒 Agent 继续一个回合，全程受每会话预算约束，预算用尽则向聊天通道告警。

它不注册工具、不发布服务、也不新增会话事件。决策读取 `pacing` 会话投影，该投影把每次 `turn/end` 折叠成最近一次回合的结局与失败事实——这是在重启后了解会话最后一次回合的官方方式（同步事件读取已被弃用）。面向 provider 的限流措辞留在核心包的 `limit.ts`，让该词汇只有一个归属地。

## 自动关机

关停闸门归 `supervisor-ops`，对闸门结果采取行动归 `@deepseek-ai/dsh-supervisor-shutdown`。闸门是运维方的策略：`shutdown.enabled`、`shutdown.command` 与 `shutdown.gracePeriodMin` 位于运维配置中，`shutdownDecision` 针对一个触发条件给出允许或拒绝的决策及其命令，`supervisor_shutdown` 只报告该决策，不执行任何东西。关机组件通过 `ctx.supervisorOps` 消费该决策，并经 `ctx.shell` 运行返回的命令；它不重复任何策略，也不自行发明命令。

组件沿四条已配置路径武装一次尝试。`scheduleAt` 墙钟被解析为它下一次本地到点时刻；`scheduleAfterMinutes` 或 `scheduleAfterSeconds` 从插件加载起倒计时，两者都设置时秒数优先；`watchPlans` 在每份已存计划的子任务都为 `completed` 或 `skipped` 时武装，并在工作重新打开时撤销该尝试；`idleTimeoutMin` 在没有 Agent 运行的时长达标后武装。`supervisor_power` 工具按监工模型的请求武装同样的尝试——`confirm` 用于它声称已完成的工作，`schedule` 用于倒计时——并报告或取消已武装的尝试。

在尝试触发之前，`remindAtMin`（默认 10、5、1 分钟）把每个阈值按每次尝试各播报一次：写进日志，并在 `notifyChatId` 指定会话时发到聊天。新尝试若启动时已落在某阈值之内，该阈值会被标记为已播报，因此一个在十分钟窗口内武装的倒计时绝不会为已经更近的截止时间报告"还有十分钟"；重新武装会重启提醒，而不是重放已送达的那些。

`cancelCommand` 是回退路径。即使本组件什么都没武装，它也会经同一 shell 接缝运行，因为更早的一次运行可能给操作系统留下了它自己的关机定时器，而只有运维方的这条命令能停下已经在倒计时的机器。`supervisor_power cancel` 按请求走这条路，工作重新打开也会取消由计划监视器武装的尝试。`dryRun` 把组件本会运行的每条命令——关机与取消——经同一条播报路径报告出来，而不是执行它们。

安全模型由四部分组成。组件绝不发明操作系统命令：闸门返回运维方的命令，而拒绝结果根本不带命令。`enabled` 默认为 false，且除非运维方启用了关机并配置了非空命令，闸门一律拒绝，因此未配置的部署永远不可能把机器关机。闸门是在尝试触发时被询问，而不是在武装时，所以仍在 `gracePeriodMin` 之内的尝试是"尚未"而不是"拒绝"。每个条件只消费一次尝试：计划完成触发的尝试在未重新打开工作前不会再次武装，空闲触发的尝试会重置空闲时钟，因此没有任何条件会运行命令两次。

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.zh.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

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

Types: [Agent](core.zh.md) · [TerminalSessionId](terminal.zh.md) · [TerminalSessionStatus](terminal.zh.md) · [TerminalWaitReason](terminal.zh.md)

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

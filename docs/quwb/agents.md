# OpenClaw Agent 实现分析

本文档分析 OpenClaw 项目中的 Agent 实现逻辑，包括上下文引擎（Context Engine）的设计。

## 目录

- [架构概览](#架构概览)
- [核心组件](#核心组件)
- [上下文引擎](#上下文引擎)
- [Agent 运行时](#agent-运行时)
- [子 Agent 系统](#子-agent-系统)
- [上下文压缩](#上下文压缩)
- [会话管理](#会话管理)

---

## 架构概览

OpenClaw 的 Agent 系统基于 `@mariozechner/pi-coding-agent`（Pi Agent）构建，采用嵌入式运行时架构。整个系统围绕以下几个核心概念设计：

```
┌─────────────────────────────────────────────────────────────────┐
│                         OpenClaw Agent System                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │   Context    │    │     Run      │    │   Subagent   │     │
│  │   Engine     │◄───┤    Manager   │───►│   Registry   │     │
│  │              │    │              │    │              │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│         │                    │                    │             │
│         ▼                    ▼                    ▼             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │  Session     │    │   Message    │    │   Announce   │     │
│  │  Manager     │    │    Queue     │    │    Flow      │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 核心文件位置

- **Agent 核心**: `src/agents/`
  - 运行时: `pi-embedded.ts`, `pi-embedded-runner/`
  - 子 Agent: `subagent-registry.ts`, `subagent-announce.ts`
- **上下文引擎**: `src/context-engine/`
- **会话管理**: `src/config/sessions.js`
- **工具系统**: `src/agents/tools/`

---

## 核心组件

### 1. 类型定义

#### Agent 元数据 (`pi-embedded-runner/types.ts`)

```typescript
export type EmbeddedPiAgentMeta = {
  sessionId: string;
  provider: string;
  model: string;
  compactionCount?: number;
  promptTokens?: number;
  usage?: {
    input?: number;
    output?: number;
    cacheRead?: number;
    cacheWrite?: number;
    total?: number;
  };
  lastCallUsage?: {
    /* ... */
  };
};
```

#### 运行结果元数据

```typescript
export type EmbeddedPiRunMeta = {
  durationMs: number;
  agentMeta?: EmbeddedPiAgentMeta;
  aborted?: boolean;
  systemPromptReport?: SessionSystemPromptReport;
  error?: {
    kind:
      | "context_overflow"
      | "compaction_failure"
      | "role_ordering"
      | "image_size"
      | "retry_limit";
    message: string;
  };
  stopReason?: string;
  pendingToolCalls?: Array<{
    /* ... */
  }>;
};
```

### 2. 运行状态管理

**文件**: `pi-embedded-runner/runs.ts`

全局单例模式管理所有活跃的 Agent 运行状态：

```typescript
const embeddedRunState = resolveGlobalSingleton(EMBEDDED_RUN_STATE_KEY, () => ({
  activeRuns: new Map<string, EmbeddedPiQueueHandle>(),
  snapshots: new Map<string, ActiveEmbeddedRunSnapshot>(),
  waiters: new Map<string, Set<EmbeddedRunWaiter>>(),
}));
```

核心功能：

- `queueEmbeddedPiMessage()`: 向正在运行的 Agent 排队消息
- `abortEmbeddedPiRun()`: 中止指定或所有运行
- `isEmbeddedPiRunActive()`: 检查会话是否有活跃运行
- `waitForEmbeddedPiRunEnd()`: 等待运行完成

---

## 上下文引擎

上下文引擎是 OpenClaw 的可插拔上下文管理系统，允许第三方插件实现自定义的上下文组装、压缩和维护逻辑。

### 核心接口

**文件**: `src/context-engine/types.ts`

```typescript
export interface ContextEngine {
  readonly info: ContextEngineInfo;

  // 初始化
  bootstrap?(params: {
    sessionId: string;
    sessionKey?: string;
    sessionFile: string;
  }): Promise<BootstrapResult>;

  // 维护（transcript 重写等）
  maintain?(params: {
    sessionId: string;
    sessionKey?: string;
    sessionFile: string;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<ContextEngineMaintenanceResult>;

  // 消息摄取
  ingest(params: {
    sessionId: string;
    sessionKey?: string;
    message: AgentMessage;
    isHeartbeat?: boolean;
  }): Promise<IngestResult>;

  // 批量摄取
  ingestBatch?(params: { /* ... */ }): Promise<IngestBatchResult>;

  // 回调后处理
  afterTurn?(params: { /* ... */ }): Promise<void>;

  // 上下文组装
  assemble(params: {
    sessionId: string;
    sessionKey?: string;
    messages: AgentMessage[];
    tokenBudget?: number;
    model?: string;
    prompt?: string;
  }): Promise<AssembleResult>;

  // 上下文压缩
  compact(params: {
    sessionId: string;
    sessionKey?: string;
    sessionFile: string;
    tokenBudget?: number;
    force?: boolean;
    currentTokenCount?: number;
    compactionTarget?: "budget" | "threshold";
    customInstructions?: string;
    runtimeContext?: ContextEngineRuntimeContext;
  }): Promise<CompactResult>;

  // 子 Agent 生命周期
  prepareSubagentSpawn?(params: { /* ... */ }): Promise<SubagentSpawnPreparation | undefined>;
  onSubagentEnded?(params: { /* ... */ }): Promise<void>;

  // 资源清理
  dispose?(): Promise<void>;
}
```

### 结果类型

```typescript
// 组装结果
export type AssembleResult = {
  messages: AgentMessage[];
  estimatedTokens: number;
  systemPromptAddition?: string;
};

// 压缩结果
export type CompactResult = {
  ok: boolean;
  compacted: boolean;
  reason?: string;
  result?: {
    summary?: string;
    firstKeptEntryId?: string;
    tokensBefore: number;
    tokensAfter?: number;
    details?: unknown;
  };
};

// Transcript 重写结果
export type TranscriptRewriteResult = {
  changed: boolean;
  bytesFreed: number;
  rewrittenEntries: number;
  reason?: string;
};
```

### 引擎注册表

**文件**: `src/context-engine/registry.ts`

全局单例注册表，支持多引擎共存：

```typescript
type ContextEngineRegistryState = {
  engines: Map<
    string,
    {
      factory: ContextEngineFactory;
      owner: string;
    }
  >;
};

export function registerContextEngineForOwner(
  id: string,
  factory: ContextEngineFactory,
  owner: string,
  opts?: RegisterContextEngineForOwnerOptions,
): ContextEngineRegistrationResult;

export function registerContextEngine(
  id: string,
  factory: ContextEngineFactory,
): ContextEngineRegistrationResult;
```

解析顺序：

1. `config.plugins.slots.contextEngine`（显式覆盖）
2. 默认 slot 值 ("legacy")

### Legacy 上下文引擎

**文件**: `src/context-engine/legacy.ts`

向后兼容的默认实现：

```typescript
export class LegacyContextEngine implements ContextEngine {
  readonly info: ContextEngineInfo = {
    id: "legacy",
    name: "Legacy Context Engine",
    version: "1.0.0",
  };

  async ingest(): Promise<IngestResult> {
    // No-op: SessionManager 处理消息持久化
    return { ingested: false };
  }

  async assemble(params: { messages: AgentMessage[] }): Promise<AssembleResult> {
    // Pass-through: 现有 sanitize/validate/limit 管道
    return { messages: params.messages, estimatedTokens: 0 };
  }

  async compact(params): Promise<CompactResult> {
    return await delegateCompactionToRuntime(params);
  }
}
```

### 初始化

**文件**: `src/context-engine/init.ts`

```typescript
export function ensureContextEnginesInitialized(): void {
  if (initialized) return;
  initialized = true;
  registerLegacyContextEngine();
}
```

---

## Agent 运行时

### 主运行流程

**文件**: `pi-embedded-runner/run.ts` (核心入口)

运行流程概览：

```
┌─────────────────────────────────────────────────────────────┐
│  1. Pre-flight Checks                                        │
│     - 认证配置文件状态                                        │
│     - 上下文窗口检查                                          │
│     - 模型降级备选                                            │
├─────────────────────────────────────────────────────────────┤
│  2. Session Setup                                            │
│     - 打开/创建 SessionManager                               │
│     - Bootstrap context engine                               │
│     - 初始化运行时插件                                        │
├─────────────────────────────────────────────────────────────┤
│  3. Context Assembly                                         │
│     - 从 SessionManager 读取消息                             │
│     - 历史限制 (dmHistoryLimit, historyLimit)                │
│     - ContextEngine.assemble()                               │
├─────────────────────────────────────────────────────────────┤
│  4. Run Attempts (Loop on overflow)                          │
│     ┌─────────────────────────────────────────────────────┐ │
│     │  a. Build agent config                              │ │
│     │  b. Execute run (streaming)                         │ │
│     │  c. Handle context overflow → compact → retry       │ │
│     └─────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  5. Post-run Processing                                      │
│     - ContextEngine.afterTurn()                              │
│     - 更新 sessions.json                                     │
│     - 触发内部 hooks                                          │
└─────────────────────────────────────────────────────────────┘
```

### 运行尝试

**文件**: `pi-embedded-runner/run/attempt.ts`

单次运行尝试的核心逻辑：

```typescript
export async function attemptEmbeddedPiRun(params: {
  // ... 大量配置参数
}): Promise<EmbeddedPiRunResult>;
```

关键步骤：

1. **上下文准备**: 消息验证、角色顺序检查
2. **Agent 构建**: 系统提示词、工具配置、沙箱设置
3. **流式执行**: 使用 `agent.run()` 处理用户输入
4. **消息处理**: 工具调用、内容块解析、消息去重

### 上下文窗口保护

**文件**: `agents/context-window-guard.ts`

```typescript
export const CONTEXT_WINDOW_HARD_MIN_TOKENS = 16_000;
export const CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32_000;

export function resolveContextWindowInfo(params: {
  cfg: OpenClawConfig | undefined;
  provider: string;
  modelId: string;
  modelContextWindow?: number;
  defaultTokens: number;
}): ContextWindowInfo;

export function evaluateContextWindowGuard(params: {
  info: ContextWindowInfo;
  warnBelowTokens?: number;
  hardMinTokens?: number;
}): ContextWindowGuardResult;
```

上下文窗口来源优先级：

1. `config.models.providers[<provider>].models[<modelId>].contextWindow`
2. 模型内置的 `modelContextWindow`
3. `config.agents.defaults.contextTokens` (上限)
4. 默认值

### 历史限制

**文件**: `pi-embedded-runner/history.ts`

```typescript
export function limitHistoryTurns(
  messages: AgentMessage[],
  limit: number | undefined,
): AgentMessage[];

export function getHistoryLimitFromSessionKey(
  sessionKey: string | undefined,
  config: OpenClawConfig | undefined,
): number | undefined;
```

支持：

- DM 会话: `dmHistoryLimit` + per-DM `dms[<userId>].historyLimit`
- 频道会话: `historyLimit`

---

## 子 Agent 系统

### 子 Agent 注册表

**文件**: `agents/subagent-registry.ts`

管理子 Agent 的生命周期：

```typescript
export type SubagentSessionEntry = {
  sessionId: string;
  sessionKey: string;
  sessionFile: string;
  parentSessionKey: string;
  createdAt: number;
  expiresAt: number | null;
  ttlMs: number | null;
};

export async function registerSubagentSession(params: {
  parentSessionKey: string;
  childSessionKey: string;
  ttlMs?: number;
}): Promise<SubagentRegistryEntry>;

export async function unregisterSubagentSession(params: {
  childSessionKey: string;
  reason?: SubagentEndReason;
}): Promise<void>;
```

### 子 Agent 生命周期事件

**文件**: `agents/subagent-lifecycle-events.ts`

```typescript
export const SUBAGENT_ENDED_REASON_COMPLETE = "subagent-complete";
export const SUBAGENT_ENDED_REASON_ERROR = "subagent-error";
export const SUBAGENT_ENDED_REASON_KILLED = "subagent-killed";
export const SUBAGENT_ENDED_REASON_SESSION_RESET = "session-reset";
export const SUBAGENT_ENDED_REASON_SESSION_DELETE = "session-delete";

export type SubagentLifecycleEndedOutcome =
  | "ok"
  | "error"
  | "timeout"
  | "killed"
  | "reset"
  | "deleted";
```

### 子 Agent 通告流程

**文件**: `agents/subagent-announce.ts`

处理子 Agent 结果向父 Agent 的报告：

```typescript
export async function runSubagentAnnounceFlow(params: {
  parentSessionKey: string;
  childSessionKey: string;
  childRunResult: EmbeddedPiRunResult;
  customAnnounceMessage?: string;
  childAgentInternalEvents?: AgentInternalEvent[];
  deliveryContext?: DeliveryContext;
}): Promise<SubagentRunOutcome>;
```

关键功能：

- 捕获子 Agent 完成回复
- 格式化内部事件供提示
- 幂等性处理 (`announceIdempotencyKey`)
- 消息去重

### 深度限制

防止无限嵌套：

```typescript
const DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 5;

// 从 session key 解析当前深度
function resolveSubagentDepthFromSessionKey(sessionKey: string): number;
```

---

## 上下文压缩

### 压缩入口

**文件**: `pi-embedded-runner/compact.ts`

```typescript
export async function compactEmbeddedPiSession(params: {
  sessionFile: string;
  sessionId?: string;
  sessionKey?: string;
  tokenBudget?: number;
  force?: boolean;
  currentTokenCount?: number;
  customInstructions?: string;
  workspaceDir?: string;
  provider?: string;
  model?: string;
  runMeta?: EmbeddedPiRunMeta;
}): Promise<EmbeddedPiCompactResult>;
```

压缩策略：

1. **阈值触发**: 默认 200k tokens 或模型上限的 80%
2. **目标收敛**: 尝试压缩到预算的 50% 或更少
3. **LLM 摘要**: 生成历史对话摘要
4. **分支管理**: 从最后一个保留消息的父节点分支

### 运行时委托

**文件**: `context-engine/delegate.ts`

允许第三方引擎使用内置压缩逻辑：

```typescript
export async function delegateCompactionToRuntime(
  params: Parameters<ContextEngine["compact"]>[0],
): Promise<CompactResult> {
  const { compactEmbeddedPiSessionDirect } =
    await import("../agents/pi-embedded-runner/compact.runtime.js");
  // ...
}
```

### Transcript 重写

**文件**: `pi-embedded-runner/transcript-rewrite.ts`

安全的分支并重新追加：

```typescript
export function rewriteTranscriptEntriesInSessionManager(params: {
  sessionManager: SessionManagerLike;
  replacements: TranscriptRewriteReplacement[];
}): TranscriptRewriteResult;

export async function rewriteTranscriptEntriesInSessionFile(params: {
  sessionFile: string;
  sessionId?: string;
  sessionKey?: string;
  request: TranscriptRewriteRequest;
}): Promise<TranscriptRewriteResult>;
```

工作原理：

1. 找到第一个需要重写的消息
2. 从该消息的父节点分支
3. 重新追加后续消息（应用替换）
4. 发送 transcript 更新事件

### 溢出恢复循环

**文件**: `pi-embedded-runner/run.overflow-compaction.ts`

当上下文溢出时的自动恢复：

```typescript
export async function runWithOverflowCompaction(params: { // ... }): Promise<EmbeddedPiRunResult>;
```

流程：

1. 尝试正常运行
2. 如果溢出 → 触发压缩 → 重试
3. 最多重试 2 次
4. 记录压缩次数到 `agentMeta.compactionCount`

---

## 会话管理

### SessionManager

基于 `@mariozechner/pi-coding-agent` 的 `SessionManager`：

```typescript
import { SessionManager, createAgentSession } from "@mariozechner/pi-coding-agent";

// 打开现有会话
const sessionManager = SessionManager.open(sessionFile);

// 创建新会话
const { sessionManager, sessionId, sessionFile } = await createAgentSession({
  sessionsDir,
  sessionKey,
});
```

### 会话存储

**文件**: `src/config/sessions.js`

```typescript
export async function loadSessionStore(): Promise<SessionStore>;
export async function updateSessionStore(
  sessionId: string,
  updates: Partial<SessionEntry>,
): Promise<void>;
export function resolveStorePath(sessionsDir: string): string;
```

### Session Key 解析

```typescript
// 格式: "agent:<provider>:<kind>:<userId>"
// 例如: "agent:telegram:dm:123456789"
//       "agent:discord:channel:123/456"

export function resolveAgentIdFromSessionKey(sessionKey: string): string;
export function resolveMainSessionKey(sessionKey: string): string;
```

### 内部事件

**文件**: `agents/internal-events.ts`

```typescript
export type AgentInternalEvent =
  | { type: "subagent_spawned" /* ... */ }
  | { type: "subagent_ended" /* ... */ }
  | { type: "cron_scheduled" /* ... */ }
  | { type: "cron_removed" /* ... */ }
  | { type: "context_compacted" /* ... */ };

export function formatAgentInternalEventsForPrompt(events: AgentInternalEvent[]): string;
```

---

## 设计特点

### 1. 可扩展性

- **插件化上下文引擎**: 第三方可以注册自定义引擎
- **Hook 系统**: 支持插件在各生命周期点注入逻辑
- **工具扩展**: 通过 `api.addTool()` 添加自定义工具

### 2. 向后兼容

- **Legacy 适配器**: `wrapContextEngineWithSessionKeyCompat()` 自动处理旧 API
- **渐进式迁移**: 新引擎可选注册，默认使用 legacy

### 3. 容错性

- **重试机制**: 溢出恢复循环、认证故障转移
- **幂等性**: announce flow 使用 `announceIdempotencyKey`
- **锁机制**: `acquireSessionWriteLock()` 防止并发写入

### 4. 可观测性

- **诊断日志**: `diagnosticLogger` 详细记录运行状态
- **事件系统**: `emitSessionTranscriptUpdate()`, `onAgentEvent()`
- **快照**: `ActiveEmbeddedRunSnapshot` 用于调试

---

## 配置项

### Agent 限制

**文件**: `src/config/agent-limits.ts`

```typescript
const DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH = 5;
const DEFAULT_SUBAGENT_MAX_PARALLEL = 3;
const DEFAULT_SUBAGENT_DEFAULT_TTL_MS = 4 * 60 * 60 * 1000; // 4h
```

### 默认值

**文件**: `src/agents/defaults.ts`

```typescript
export const DEFAULT_MODEL = "claude-opus-4";
export const DEFAULT_PROVIDER = "anthropic";
export const DEFAULT_CONTEXT_TOKENS = 200_000;
```

### 上下文窗口

**文件**: `src/agents/context-window-guard.ts`

```typescript
export const CONTEXT_WINDOW_HARD_MIN_TOKENS = 16_000;
export const CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32_000;
```

---

## 相关文档

- [配置参考](/gateway/configuration-reference)
- [插件架构](/plugins/architecture)
- [上下文引擎概念](/concepts/context-engine)
- [压缩概念](/concepts/compaction)
- [上下文概念](/concepts/context)

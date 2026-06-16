# Mage AI 前端 Editor 通信：运行保护分支逻辑分析

> 本文聚焦 Editor 通信中「运行保护分支」的三个核心问题：
> 1. **先保存再运行**的操作顺序与各分支的精确语义
> 2. **运行中且未忽略时只刷新状态不发送执行消息**的规则
> 3. `ignoreAlreadyRunning` 参数的使用场景

---

## 一、双层函数结构

用户触发代码执行时，经过两层函数：

```
runBlock()           ← 外层：决定是否先保存
    └─► runBlockOrig()  ← 内层：决定是否发送执行消息 + 刷新状态
```

### 1.1 runBlock（外层）—— 保存决策

`mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` L2667-L2704

```typescript
const runBlock = useCallback((payload, options) => {
  const { block } = payload;

  if (disablePipelineEditAccess || options?.skipUpdating) {
    // 分支 A：跳过保存，直接执行
    return runBlockOrig(payload, options);
  } else {
    // 分支 B：先保存再执行
    return savePipelineContent({
      block: { outputs: [], uuid: block.uuid },
    }, {
      contentOnly: true,
    })?.then(() => runBlockOrig(payload));
  }
}, [disablePipelineEditAccess, runBlockOrig, savePipelineContent]);
```

### 1.2 runBlockOrig（内层）—— 执行决策

`mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` L2571-L2665

```typescript
const runBlockOrig = useCallback((payload, options) => {
  const { block, code, ignoreAlreadyRunning, ... } = payload;
  const { uuid } = block;

  // ★ 核心：检查 block 是否已在运行
  const isAlreadyRunning = runningBlocks.find(({ uuid: uuid2 }) => uuid === uuid2);

  if (!isAlreadyRunning || ignoreAlreadyRunning) {
    // 分支 ①：未运行 或 忽略运行状态 → 发送执行消息
    sendMessage(JSON.stringify({
      ...sharedWebsocketData,
      code,
      uuid,
      type: block.type,
      pipeline_uuid: pipeline?.uuid,
      // ...其他字段
    }));

    setMessages(prev => { delete prev[uuid]; return prev; });  // 清空旧输出
    setTextareaFocused(false);
    setRunningBlocks(prev => prev.find(…)? prev : prev.concat(block));  // 标记运行中
    setOutputsToSaveByBlockUUID(prev => ({ ...prev, [block.uuid]: true }));
  }
  // 分支 ②：运行中且未忽略 → 不发送执行消息（上面整个 if 被跳过）

  // ★ 无论哪个分支，都会执行：
  if (!options?.skipUpdating) {
    fetchPipeline();  // 刷新依赖图状态
  }
}, [fetchPipeline, pipeline, runningBlocks, sendMessage, ...]);
```

---

## 二、先保存再运行的完整决策树

```
用户按下 Ctrl+Enter / Cmd+Enter
            │
            ▼
      ┌──────────┐
      │ runBlock │
      └─────┬────┘
            │
    ┌───────┴───────────────────────┐
    │ disablePipelineEditAccess     │
    │ || options?.skipUpdating ?    │
    └───┬───────────────────┬───────┘
      是│                   │否
        ▼                   ▼
  ┌──────────┐    ┌──────────────────────┐
  │ 分支 A   │    │ 分支 B               │
  │ 不保存   │    │ savePipelineContent  │
  │ 直接执行 │    │ ({contentOnly:true}) │
  └────┬─────┘    │   ↓                  │
       │          │ updatePipeline(PUT)  │
       │          │   ↓                  │
       │          │ .then()              │
       │          └──────┬───────────────┘
       │                 │
       └────────┬────────┘
                ▼
      ┌──────────────┐
      │ runBlockOrig │
      └──────┬───────┘
             │
    ┌────────┴──────────────────────┐
    │ isAlreadyRunning &&           │
    │ !ignoreAlreadyRunning ?       │
    └───┬──────────────────┬────────┘
      是│                  │否
        ▼                  ▼
  ┌──────────┐     ┌──────────────────┐
  │ 分支 ②   │     │ 分支 ①           │
  │ 运行中   │     │ 未运行/忽略运行   │
  │          │     │                  │
  │ ✗ 不发送 │     │ ✓ sendMessage()  │
  │   执行   │     │ ✓ 清空旧输出     │
  │   消息   │     │ ✓ 标记运行中     │
  │ ✓ 仅    │     │ ✓ 标记输出待保存  │
  │   刷新  │     │ ✓ 刷新依赖图     │
  │   依赖  │     └──────────────────┘
  │   图    │
  │ ✓ 刷新  │
  └──────────┘
```

---

## 三、运行中且未忽略：只刷新状态不发送执行消息

### 3.1 规则详解

当 `runBlockOrig` 被调用时：

```typescript
const isAlreadyRunning = runningBlocks.find(({ uuid: uuid2 }) => uuid === uuid2);

if (!isAlreadyRunning || ignoreAlreadyRunning) {
    // ← 只有这里才发送 WebSocket 执行消息
    sendMessage(...);
    setMessages(...);
    setRunningBlocks(...);
    setOutputsToSaveByBlockUUID(...);
}
// ← isAlreadyRunning 为真 且 ignoreAlreadyRunning 为假时
//    整个 if 块被跳过，不会发送任何执行消息

if (!options?.skipUpdating) {
    fetchPipeline();  // ← 但这里始终执行
}
```

**行为对照**：

| 操作 | isAlreadyRunning=true 且 ignoreAlreadyRunning=false | 其他情况 |
|------|:---:|:---:|
| `sendMessage`（发 WebSocket 执行请求） | ✗ 不发送 | ✓ 发送 |
| `setMessages`（清空旧输出） | ✗ 不清空 | ✓ 清空 |
| `setRunningBlocks`（标记运行中） | ✗ 不追加 | ✓ 追加 |
| `setOutputsToSaveByBlockUUID`（标记输出待保存） | ✗ 不标记 | ✓ 标记 |
| `fetchPipeline`（刷新依赖图） | ✓ 刷新 | ✓ 刷新 |

### 3.2 设计意图

**避免重复执行**：如果 block 正在运行中（`runningBlocks` 包含该 uuid），再次按运行键不会向后端发送新的执行请求。这防止了：
- 后端内核同一 block 代码被重复 execute
- 输出消息交错混乱
- 不必要的资源消耗

**但仍然刷新依赖图**：`fetchPipeline()` 调用 `api.pipelines.detail()` 获取最新管道状态，更新 block 之间的依赖关系和执行状态。这确保用户即使没有重新触发执行，也能看到依赖图的最新状态。

### 3.3 `fetchPipeline` 的行为

`mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` L306-L329

```typescript
const { data, mutate: fetchPipeline } = api.pipelines.detail(
    pipelineUUID,
    {
        include_block_pipelines: true,
        includes_outputs: isEmptyObject(messages) || ...,
        ...(includeSparkOutputs ? { includes_outputs_spark: true } : {}),
    },
    {
        refreshInterval: null,           // 不自动刷新
        revalidateOnFocus: false,        // 切回窗口不自动刷新
    },
);
```

`fetchPipeline` 是 react-query 的 `mutate` 函数，触发一次 `GET /api/pipelines/:uuid` 请求。查询参数中的 `includes_outputs` 控制是否在响应中包含 block 输出数据。

---

## 四、`ignoreAlreadyRunning` 的使用场景

### 4.1 参数定义

`ignoreAlreadyRunning` 是 `runBlock` / `runBlockOrig` 的 `payload` 中的可选布尔字段，默认为 `undefined`（falsy）。

### 4.2 调用方分析

经全仓库搜索，`ignoreAlreadyRunning: true` 只在一个场景中显式传入：

**ChartBlock（图表 Widget）**：

`mage_ai/frontend/components/ChartBlock/index.tsx` L229-L240

```typescript
savePipelineContent().then(() => {
    runBlock({
        block: widget,
        code: content,
        ignoreAlreadyRunning: true,    // ★ 图表允许重复执行
        runUpstream: !!upstreamBlocks.find(
            (uuid) => ![StatusTypeEnum.EXECUTED, StatusTypeEnum.UPDATED]
                        .includes(blocksMapping[uuid]?.status),
        ),
    });
});
```

**为什么图表需要 `ignoreAlreadyRunning: true`？**

图表 block 是下游消费者，依赖上游数据块的输出。当上游 block 执行完毕后，需要重新运行图表来刷新可视化。此时图表可能正在运行中（前一次渲染还没完成），但上游数据已更新，必须立即重新执行以反映最新数据。如果被"运行中"保护拦截，图表就无法及时更新。

### 4.3 其他调用方

| 调用方 | ignoreAlreadyRunning | 原因 |
|--------|:---:|------|
| **CodeBlock**（普通代码块） | 不传（falsy） | 普通 block 不应重复执行 |
| **ChartBlock**（图表） | `true` | 需要响应上游数据更新，允许覆盖执行 |
| **CustomTemplates/TemplateDetail** | 不传（falsy） | 模板执行不应重复 |

---

## 五、完整分支矩阵

综合 `runBlock` 外层和 `runBlockOrig` 内层的所有条件：

| # | disablePipelineEditAccess | skipUpdating | isAlreadyRunning | ignoreAlreadyRunning | 保存 | 发送执行消息 | 刷新依赖图 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | ✗ | ✗ | ✗ | ✗ | ✓ 先保存 | ✓ | ✓ |
| 2 | ✗ | ✗ | ✓ | ✗ | ✓ 先保存 | ✗ | ✓ |
| 3 | ✗ | ✗ | ✓ | ✓ | ✓ 先保存 | ✓ | ✓ |
| 4 | ✓ | - | ✗ | ✗ | ✗ | ✓ | ✓ |
| 5 | ✓ | - | ✓ | ✗ | ✗ | ✗ | ✓ |
| 6 | - | ✓ | ✗ | ✗ | ✗ | ✓ | ✗ |
| 7 | - | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |

**关键行解读**：

- **行 2**（最常被忽视的分支）：正常模式下，block 正在运行且不忽略 → **保存了内容但不发送执行消息**，只刷新依赖图。这就是"保存但不一定发送执行消息"的典型场景。
- **行 5**：禁止编辑模式下，block 正在运行且不忽略 → **不保存也不发送**，只刷新依赖图。
- **行 7**：显式 skipUpdating + 运行中 → **既不保存也不发送也不刷新**，完全静默。

---

## 六、时序图：正常执行 vs 运行中重复触发

### 场景 A：正常执行（block 未运行）

```
用户         runBlock      savePipelineContent   runBlockOrig   WebSocket   后端       前端onMessage
 │              │                │                   │              │          │            │
 │──运行───────►│                │                   │              │          │            │
 │              │──保存─────────►│                   │              │          │            │
 │              │                │──PUT /api/pip────►│              │          │            │
 │              │                │                   │──sendMessage►│          │            │
 │              │                │                   │              │──execute►│            │
 │              │                │                   │              │          │──输出──────►│
 │              │                │                   │              │          │            │──渲染
 │              │                │                   │──fetchPipe──►│          │            │
```

### 场景 B：block 运行中 + ignoreAlreadyRunning=false

```
用户         runBlock      savePipelineContent   runBlockOrig   WebSocket   后端       前端onMessage
 │              │                │                   │              │          │            │
 │──运行───────►│                │                   │              │          │            │
 │              │──保存─────────►│                   │              │          │            │
 │              │                │──PUT /api/pip────►│              │          │            │
 │              │                │                   │              │          │            │
 │              │                │                   │  isAlreadyRunning=true     │            │
 │              │                │                   │  ignoreAlreadyRunning=false│            │
 │              │                │                   │──✗ 不发送──► │          │            │
 │              │                │                   │              │          │            │
 │              │                │                   │──fetchPipe──►│          │            │
 │              │                │                   │  (仅刷新依赖图)           │            │
```

> **注意**：场景 B 中，`savePipelineContent` 仍然执行了 PUT 请求保存内容——用户的代码编辑被持久化了。但执行请求没有发出，因为 block 正在运行中。当当前运行完成后（`execution_state=idle`），前端将该 block 从 `runningBlocks` 移除，用户可以再次触发执行。

---

## 七、关键文件索引

| 关注点 | 仓库相对路径 | 关键行号 |
|--------|-------------|---------|
| runBlock（保存决策层） | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2667-L2704 |
| runBlockOrig（执行决策层） | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2571-L2665 |
| isAlreadyRunning 判断 | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2605-L2607 |
| fetchPipeline 定义 | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L306-L329 |
| ChartBlock 使用 ignoreAlreadyRunning=true | `mage_ai/frontend/components/ChartBlock/index.tsx` | L229-L240 |
| savePipelineContent | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L1187-L1446 |
| executePipeline（无条件保存） | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2504-L2514 |

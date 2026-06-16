# 监控与 Stats 深度分析报告

## 一、体系架构总览

Mage AI 的监控与状态同步体系由四层机制组成，各自承担不同的职责边界：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        前端应用层 (Next.js)                          │
│                                                                     │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ 前端轮询机制   │  │  WebSocket   │  │  SSE (EventSource)   │  │
│  │ (HTTP + SWR)   │  │  双向通信    │  │  单向数据流          │  │
│  └───────┬────────┘  └──────┬───────┘  └──────────┬───────────┘  │
│          │                  │                       │              │
│  ┌───────▼────────┐         │                       │              │
│  │ 数据请求缓存   │         │                       │              │
│  │  (SWR Cache)   │         │                       │              │
│  └────────────────┘         │                       │              │
└───────────┬──────────────────┼───────────────────────┼──────────────┘
            │ HTTP API        │ ws://                 │ http://
            │                 │                       │
┌───────────▼──────────────────▼───────────────────────▼──────────────┐
│                        服务端 (Tornado)                              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │ REST API     │  │ WebSocket    │  │ SSE Handler          │     │
│  │ MonitorStat  │  │ Server       │  │ EventStreamHandler   │     │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘     │
│         │                 │                       │                 │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────────▼───────────┐     │
│  │ 数据库       │  │ Jupyter      │  │ 执行结果队列         │     │
│  │ PipelineRun  │  │ Kernel       │  │ (faster_fifo Queue)  │     │
│  │ BlockRun     │  │              │  │                      │     │
│  └──────────────┘  └──────────────┘  └──────────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
```

四种状态同步机制的核心区别：

| 维度 | 前端轮询 | 数据请求缓存 | WebSocket | SSE 事件流 |
|-----|---------|-------------|-----------|-----------|
| 通信方向 | 单向（客户端拉取） | 本地读写 | 双向 | 单向（服务端推送） |
| 实时性 | 延迟 = 轮询间隔 | 瞬时（本地缓存） | 近实时 | 近实时（~0.1s） |
| 数据范围 | 历史统计、列表数据 | 已请求过的数据 | 交互式执行输出 | 执行结果流 |
| 状态所有者 | 服务端数据库 | 前端内存 | 服务端内存 | 服务端队列 |
| 持久化 | 持久化 | 不持久化 | 不持久化 | 不持久化 |
| 重连/失效 | 自动重新验证 | 组件卸载后保留 | 自动重连（10次/3s） | 自动重连（指数退避） |
| 适用场景 | 统计图表、历史列表 | 重复查询优化 | 块执行、管道执行、终端 | 代码执行输出流 |

---

## 二、Stats 监控刷新时机详细梳理

### 2.1 监控详情页刷新设置核对

三个监控详情页均位于 `mage_ai/frontend/pages/pipelines/[pipeline]/monitors/` 目录下。

#### 2.1.1 管道运行监控页（index.tsx）

**刷新配置核对**：

```typescript
// monitor_stats 查询 - 无任何 SWR 配置
const { data: dataMonitor } = api.monitor_stats.detail(
  'pipeline_run_count',
  { pipeline_uuid: pipeline?.uuid },
);

// pipelines 查询 - 仅设置 revalidateOnFocus: false
const { data: dataPipeline } = api.pipelines.detail(pipelineUUID, {
  includes_content: false,
  includes_outputs: false,
}, {
  revalidateOnFocus: false,
});
```

| 配置项 | 值 | 来源 |
|-------|----|------|
| refreshInterval | 未设置（SWR 默认：0，即不轮询） | monitor_stats 查询未传 swrOptions |
| revalidateOnFocus | monitor_stats 未设置（SWR 默认：true），pipelines 设为 false | 代码中可核对 |
| 手动刷新 | 无 mutate 暴露 | 未解构 mutate |
| 触发刷新时机 | 首次挂载 + 窗口聚焦 | SWR 默认行为 |

#### 2.1.2 块运行监控页（block-runs.tsx）

**刷新配置核对**：

```typescript
// 解构了 mutate: fetchStats，用于手动刷新
const { data: dataMonitor, mutate: fetchStats } =
  api.monitor_stats.detail('block_run_count', monitorStatQuery);
```

| 配置项 | 值 | 来源 |
|-------|----|------|
| refreshInterval | 未设置（SWR 默认：0，即不轮询） | monitor_stats 查询未传 swrOptions |
| revalidateOnFocus | 未设置（SWR 默认：true） | monitor_stats 查询未传 swrOptions |
| 手动刷新 | ✅ 有 mutate: fetchStats | 切换调度 Select 时调用 |
| 触发刷新时机 | 首次挂载 + 窗口聚焦 + 切换调度器 | Select onChange 触发 fetchStats() |

#### 2.1.3 块运行时间监控页（block-runtime.tsx）

**刷新配置核对**：

```typescript
const { data: dataMonitor, mutate: fetchStats } =
  api.monitor_stats.detail('block_run_time', monitorStatQuery);

useEffect(() => {
  fetchStats(pipelineSchedule);
}, [fetchStats, pipelineSchedule]);
```

| 配置项 | 值 | 来源 |
|-------|----|------|
| refreshInterval | 未设置（SWR 默认：0，即不轮询） | monitor_stats 查询未传 swrOptions |
| revalidateOnFocus | 未设置（SWR 默认：true） | monitor_stats 查询未传 swrOptions |
| 手动刷新 | ✅ 有 mutate: fetchStats | 切换调度 + useEffect 触发 |
| 触发刷新时机 | 首次挂载 + 窗口聚焦 + 切换调度器 + useEffect | pipelineSchedule 变化时触发 |

#### 2.1.4 三页面对比总结

| 页面 | refreshInterval | revalidateOnFocus | 手动 mutate | 自动刷新 |
|-----|-----------------|-------------------|------------|---------|
| 管道运行监控 | ❌ 未设置（默认 0） | pipelines: false<br>monitor_stats: 未设置（默认 true） | ❌ 无 | 仅首次 + 聚焦 |
| 块运行监控 | ❌ 未设置（默认 0） | ❌ 未设置（默认 true） | ✅ 有（切换调度） | 首次 + 聚焦 + 切换调度 |
| 块运行时间监控 | ❌ 未设置（默认 0） | ❌ 未设置（默认 true） | ✅ 有（切换调度 + useEffect） | 首次 + 聚焦 + 切换调度 |

**关键结论**：三个监控详情页均**无自动轮询刷新**，只有首次加载和窗口聚焦时才会刷新数据。块相关页面额外支持在切换调度器时手动刷新。

### 2.2 概览页刷新设置

概览页位于 `mage_ai/frontend/pages/overview/index.tsx`。

**共享配置**：

```typescript
const SHARED_FETCH_OPTIONS = {
  refreshInterval: 60000,      // 60 秒
  revalidateOnFocus: false,    // 聚焦时不刷新
};
```

**monitor_stats 查询**：使用 `useMutation` 手动模式，**不走 SWR 自动轮询**

```typescript
const [fetchMonitorStats, { isLoading: isValidatingMonitorStats }] = useMutation(
  () => api.monitor_stats?.detailAsync(MonitorStatsEnum.PIPELINE_RUN_COUNT, ...),
  { onSuccess: (response) => setMonitorStats(stats) }
);
```

**pipeline_runs 查询**：使用 SWR 自动轮询，60 秒间隔

```typescript
const { data: dataPipelineRuns } = api.pipeline_runs.list(
  { ...查询参数... },
  { ...SHARED_FETCH_OPTIONS },   // 60s refreshInterval
);
```

**刷新触发时机**：
1. 组件首次挂载时调用 `fetchMonitorStats()`（useEffect 中）
2. 切换时间周期 tab 时调用 `fetchMonitorStats()`
3. pipeline_runs 每 60 秒自动轮询一次
4. **注意**：monitor_stats 本身不自动轮询，依赖手动调用

### 2.3 系统状态延迟配置核对

#### 2.3.1 默认值链路

**useDelayFetch 默认值**（`mage_ai/frontend/api/utils/useDelayFetch.ts`）：

```typescript
const {
  condition,
  delay,
  pauseFetch,
} = opts || {
  condition: undefined,
  delay: 3000,   // 默认 3 秒
};
```

| 参数 | 默认值 |
|-----|--------|
| delay | 3000ms（3 秒） |
| condition | undefined（即无条件，时间到就开始） |
| pauseFetch | 无（默认不暂停） |

**useStatus 默认值**（`mage_ai/frontend/utils/models/status/useStatus.ts`）：

```typescript
function useStatus({
  delay = 7000,        // 默认 7 秒，覆盖 useDelayFetch 的 3 秒
  pauseFetch,
  refreshInterval,
  revalidateOnFocus,
} = {}) {
  const { data: serverStatus } = useDelayFetch(api.statuses.list, {}, {
    refreshInterval,
    revalidateOnFocus,
  }, {
    delay,          // 传入 7000，覆盖 useDelayFetch 的 3000 默认值
    condition: () => !pauseFetch,
  });
}
```

| 参数 | 默认值 | 说明 |
|-----|--------|------|
| delay | 7000ms（7 秒） | 覆盖 useDelayFetch 的 3 秒默认值 |
| pauseFetch | 未设置 | 通过 condition 控制 |
| refreshInterval | 未设置 | 不传则使用 SWR 默认（0，不轮询） |
| revalidateOnFocus | 未设置 | 不传则使用 SWR 默认（true） |

#### 2.3.2 应用入口传入值

**_app.tsx（全局入口）**（`mage_ai/frontend/pages/_app.tsx`）：

```typescript
const { status } = useStatus({
  delay: 3000,                     // 覆盖默认 7000，改为 3 秒
  pauseFetch: !noValue && !noValuePermissions,  // 有权限且有值才请求
});
```

**edit.tsx（管道编辑页）**（`mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx`）：

```typescript
const { status } = useStatus();  // 不传入参数，使用默认 7 秒
```

**其他使用场景**（Header、PipelineDetail 等）：均为默认配置调用

#### 2.3.3 延迟配置总结

| 使用位置 | delay 值 | pauseFetch | refreshInterval | 说明 |
|---------|----------|------------|-----------------|------|
| useDelayFetch 默认 | 3000ms | - | - | 底层默认 |
| useStatus 默认 | 7000ms | 通过 condition 控制 | 未设置（默认 0） | 覆盖底层默认 |
| _app.tsx（全局） | 3000ms | 有权限才开始 | 未设置 | 全局状态栏，3 秒延迟 |
| edit.tsx（编辑页） | 7000ms（默认） | 未设置 | 未设置 | 使用默认值 |
| 其他页面 | 7000ms（默认） | 未设置 | 未设置 | 大部分场景 |

### 2.4 运行相关页面的刷新设置

#### 2.4.1 管道运行详情页

- **位置**：`mage_ai/frontend/pages/pipelines/[pipeline]/runs/[run]/index.tsx`
- **refreshInterval**：动态，非 idle 状态 3000ms
- **revalidateOnFocus**：true

#### 2.4.2 管道运行列表页

- **位置**：`mage_ai/frontend/pages/pipelines/[pipeline]/runs/index.tsx`
- **refreshInterval**：5000ms（固定 5 秒）
- **revalidateOnFocus**：false

### 2.5 Stats 监控刷新时机全景图

```
页面加载
   │
   ├─ 概览页
   │    ├─ monitor_stats: 手动触发（首次 + tab 切换）
   │    └─ pipeline_runs: SWR 60s 轮询 + 聚焦不刷新
   │
   ├─ 监控详情页（3 个）
   │    └─ monitor_stats: 首次加载 + 窗口聚焦（SWR 默认）
   │         ├─ 块运行/块运行时间页：切换调度器时手动刷新
   │         └─ 管道运行页：无手动刷新
   │
   ├─ 运行详情页
   │    └─ pipeline_runs: 运行中 3s 轮询 + 聚焦刷新
   │
   ├─ 运行列表页
   │    └─ pipeline_runs: 5s 轮询 + 聚焦不刷新
   │
   └─ 系统状态
        ├─ _app.tsx: 延迟 3s 后开始 + 无轮询
        └─ edit.tsx: 延迟 7s 后开始 + 无轮询
```

---

## 三、数据请求缓存机制

### 3.1 SWR 缓存架构

SWR 缓存是前端轮询的配套机制，负责缓存已请求过的数据，减少重复请求。

**核心实现**：`mage_ai/frontend/api/utils/use.ts`

**缓存键策略**：

```typescript
const url = validateID(id) ? buildUrl(resource, id) : null;
const key = url && keyInit ? keyInit : url;
```

- 默认 key = API 端点 URL（含查询参数）
- 支持自定义 key（`customOptions.key`）
- `pauseFetch` 为 true 时返回 null，SWR 跳过请求

### 3.2 缓存特性矩阵

| 特性 | 状态 | 说明 |
|-----|------|------|
| 全局共享缓存 | ✅ | 相同 URL 的组件共享同一份缓存 |
| 请求去重 | ✅ | SWR 内置 dedupingInterval（默认 2000ms） |
| 焦点重验证 | 配置化 | 监控页关闭，运行详情页开启 |
| 窗口聚焦刷新 | 配置化 | 多数监控页面关闭 |
| 手动突变（mutate） | ✅ | 支持手动触发刷新缓存 |
| 持久化缓存 | ❌ | 仅内存缓存，刷新页面后丢失 |
| 缓存 TTL | ❌ | 无过期时间配置，依赖重新验证 |
| 缓存淘汰策略 | ❌ | 无 LRU/LFU 等淘汰机制 |
| 乐观更新 | ❌ | 未使用 mutate 的乐观更新能力 |

### 3.3 缓存的状态同步边界

| 边界项 | 说明 |
|-------|------|
| **缓存生命周期** | 随页面刷新重置，无持久化 |
| **数据一致性** | 依赖重新验证，验证前可能展示旧数据 |
| **跨页面共享** | 同域名下共享，不同标签页不共享 |
| **内存占用** | 无限增长，无淘汰策略，长期使用可能内存膨胀 |
| **并发请求** | 同一 key 的并发请求会被去重合并 |
| **错误缓存** | 请求失败时保留旧数据（stale） |

### 3.4 监控页面的特殊缓存模式

概览页的 monitor_stats 采用 `useMutation` + 手动调用的模式，而非 SWR 自动管理：

```typescript
const [fetchMonitorStats, { isLoading: isValidatingMonitorStats }] = useMutation(
  () => api.monitor_stats?.detailAsync(MonitorStatsEnum.PIPELINE_RUN_COUNT, ...),
  { onSuccess: (response) => setMonitorStats(stats) }
);
```

**原因分析**：
- 时间范围切换时需要 AbortController 取消请求
- 需要更精细的加载状态控制
- 60 秒的低频刷新下，手动控制更简单

**代价**：失去了 SWR 的全局缓存、自动去重、后台重新验证等能力

---

## 四、前端轮询机制

### 4.1 轮询架构与实现

前端轮询基于 **SWR (stale-while-revalidate)** 库实现，通过封装统一的 API Hook 提供给各页面使用。

**核心实现文件**：
- `mage_ai/frontend/api/utils/use.ts` — SWR Hook 封装（useDetail / useList / useListWithParent）
- `mage_ai/frontend/api/index.ts` — API 资源自动生成器（RESOURCES_PAIRS_ARRAY 配置）

**关键封装函数**：
- `useDetail()` — 详情查询，SWR key 为资源 URL
- `useList()` — 列表查询，SWR key 为列表 URL
- `useDetailWithParent()` — 带父资源的详情查询
- `useListWithParent()` — 带父资源的列表查询

### 4.2 轮询间隔配置汇总

| 页面 | 资源 | refreshInterval | revalidateOnFocus | 说明 |
|-----|------|-----------------|-------------------|------|
| 概览页 | monitor_stats | ❌ 无（useMutation 手动） | - | 手动触发，非 SWR 自动 |
| 概览页 | pipeline_runs（失败列表） | 60000ms | false | SWR 自动轮询 |
| 管道监控页-管道运行 | monitor_stats | 未设置（默认 0） | monitor_stats: 未设置（默认 true）<br>pipelines: false | 仅首次加载 + 聚焦 |
| 管道监控页-块运行 | monitor_stats | 未设置（默认 0） | 未设置（默认 true） | 首次 + 聚焦 + 切换调度 |
| 管道监控页-块运行时间 | monitor_stats | 未设置（默认 0） | 未设置（默认 true） | 首次 + 聚焦 + 切换调度 |
| 管道运行详情页 | pipeline_runs | 非 idle 时 3000ms | true | 运行中高频轮询，空闲时停止 |
| 管道运行列表页 | pipeline_runs | 5000ms | false | 固定频率轮询 |
| 系统状态（_app） | statuses | 未设置（默认 0） | 未设置 | 延迟 3000ms 后开始 |
| 系统状态（edit） | statuses | 未设置（默认 0） | 未设置 | 延迟 7000ms 后开始 |

### 4.3 延迟拉取机制

**文件**：`mage_ai/frontend/api/utils/useDelayFetch.ts`

用于非关键路径的数据延迟加载，避免页面初始化时请求风暴：

```
页面加载
   │
   ▼
延迟等待 (delay ms, 默认 3000ms)
   │
   ├─ condition 满足 → 开始 SWR 请求
   └─ condition 不满足 → 继续等待，递归检查
```

**应用场景**：系统状态（useStatus）、内核列表（useKernel）、文件组件、Git 分支等。

**各场景延迟配置**：

| 场景 | delay 值 | 说明 |
|-----|----------|------|
| useStatus 默认 | 7000ms | 系统状态，优先级低 |
| _app.tsx useStatus | 3000ms | 全局状态栏，稍快 |
| useKernel | 5000ms | 内核列表 |
| useDelayFetch 默认 | 3000ms | 底层默认 |
| Git 分支（Header） | 3000ms（默认） | 非关键信息 |

### 4.4 轮询的状态同步边界

| 边界项 | 说明 |
|-------|------|
| **数据新鲜度** | 最差情况 = 轮询间隔，期间数据可能已变化 |
| **状态一致性** | 最终一致，轮询间隔内可能存在窗口不一致 |
| **连接断开** | 请求失败时 SWR 自动重试，但不保证数据连续 |
| **实时性上限** | 受限于轮询频率，无法实现真正的实时更新 |
| **服务端压力** | 客户端数 × 轮询频率 = 请求量，高频轮询压力大 |
| **无效请求** | 数据无变化时，轮询请求是浪费的 |

---

## 五、WebSocket 状态同步机制

### 5.1 WebSocket 服务端架构

**核心文件**：`mage_ai/server/websocket_server.py`

基于 Tornado 的 `WebSocketHandler` 实现，采用类级别的状态管理。

### 5.2 连接生命周期

```
客户端连接
   │
   ▼
on_open → 加入 clients 集合
   │
   ▼
on_message → 解析消息 → 认证 → 权限校验 → 分发执行
   │         │
   │         ├─ 单块执行 → Jupyter Kernel
   │         ├─ 管道执行 → 多进程 + Queue
   │         └─ 取消执行 → terminate 进程
   │
   ▼
send_message → 广播给所有客户端
   │
   ▼
on_close → 从 clients 移除
```

### 5.3 核心状态映射

**running_executions_mapping**（`mage_ai/server/websocket_server.py`）：

维护正在执行的消息 ID 到元数据的映射：

```python
WebSocketServer.running_executions_mapping = {
    msg_id: {
        block_type: 'data_loader',      # 块类型
        block_uuid: 'block_uuid_123',   # 块 UUID
        replicated_block: ...,          # 是否为复制块
        pipeline_uuid: 'pipe_123',      # 管道 UUID
    }
}
```

**作用**：Jupyter Kernel 的异步输出通过 `msg_id` 回调，需要此映射来关联对应的块元数据，前端才能正确定位到哪个块的输出。

**写入时机**：
- 执行单个块前
- 执行管道块前
- 管道整体执行时

### 5.4 消息处理流程

**入站消息处理**：

```
收到原始消息
   │
   ▼
JSON 解析 → Message 对象
   │
   ▼
认证校验（api_key + token）
   │
   ▼
权限校验（Pipeline edit access）
   │
   ├─ cancel_pipeline → 终止进程 + 恢复配置
   ├─ execute_pipeline → 多进程执行 + Queue 通信
   ├─ 块执行 → Jupyter Kernel.execute()
   └─ terminal → 终端消息转发
```

**出站消息广播**：

```
消息生成
   │
   ▼
should_filter_message()
   ├─ 空消息过滤（无 data/error/execution_state/type）
   └─ Jupyter widget 过滤（FloatProgress 等）
   │
   ▼
filter_out_sensitive_data() → 环境变量脱敏
   │
   ▼
简化错误堆栈 → 移除内部 Mage 方法
   │
   ▼
注入元数据 → block_uuid / pipeline_uuid / block_type
   │
   ▼
遍历所有 clients → 发送消息
```

### 5.5 管道执行的状态同步

**多进程架构**（`mage_ai/server/websocket_server.py` & `mage_ai/server/execution_manager.py`）：

```
主进程 (WebSocket)
   │
   ├─ 创建 multiprocessing.Queue
   ├─ 启动子进程 run_pipeline()
   └─ 启动 check_for_messages 协程
              │
              ▼
         轮询 Queue (0.05s 间隔)
              │
              ▼
         收到消息 → WebSocket 广播
```

**子进程**：
- 实际执行 pipeline
- 通过 `queue.put()` 发送状态消息
- 包含执行状态、输出、错误等

### 5.6 前端 WebSocket 客户端

**实现方式**：`react-use-websocket` 库（`mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx`）

**配置**：
- `reconnectAttempts: 10` — 最多重连 10 次
- `reconnectInterval: 3000` — 重连间隔 3 秒
- `shouldReconnect: () => true` — 总是尝试重连

**前端状态管理**（组件本地 state）：
- `messages` — 各块的输出消息字典 `{ [uuid]: Message[] }`
- `pipelineMessages` — 管道执行消息数组
- `runningBlocks` — 当前运行中的块列表
- `isPipelineExecuting` — 管道是否在执行中

**状态更新逻辑**：
- `execution_state === 'busy'` → 块加入 runningBlocks
- `execution_state === 'idle'` → 块移除 runningBlocks，管道执行结束时刷新管道数据

### 5.7 WebSocket 的状态同步边界

| 边界项 | 说明 |
|-------|------|
| **状态所有权** | 服务端内存中的运行时状态，不持久化 |
| **连接状态丢失** | 刷新页面/重连后，所有执行状态丢失 |
| **消息可靠性** | 无确认机制，网络不稳定时可能丢消息 |
| **广播范围** | 所有连接的客户端都收到所有消息，无房间隔离 |
| **消息顺序** | 依赖 TCP 保证顺序，应用层无序号 |
| **心跳检测** | 无 ping/pong，死连接可能长时间不察觉 |
| **状态恢复** | 重连后无法恢复历史消息和执行状态 |
| **连接数限制** | 无最大连接数控制 |
| **消息限流** | 无限速，高频输出可能阻塞 |

---

## 六、SSE 事件流机制

### 6.1 SSE 服务端架构

**核心文件**：`mage_ai/server/events/stream.py`

基于 Tornado 的异步 RequestHandler 实现 SSE（Server-Sent Events）。

### 6.2 核心实现机制

**EventStreamHandler**：

```
客户端 GET /event-streams/{uuid}
   │
   ▼
设置响应头：
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive
   │
   ▼
进入无限循环：
   │
   ├─ 从 execution_result_queue 获取队列
   │
   ├─ 非阻塞 get_nowait() 取消息
   │     │
   │     ├─ 有消息 → 包装为 EventStream → 写入 data: ...
   │     └─ 空队列 → 跳过
   │
   ├─ flush() 刷新缓冲区
   │
   └─ asyncio.sleep(0.1) → 下次循环
```

**关键参数**：
- `SLEEP_SECONDS = 0.1` — 轮询间隔 100ms
- 使用 `faster_fifo.Queue` 作为高性能队列（可选，回退到 `multiprocessing.Queue`）

### 6.3 队列管理

**核心文件**：`mage_ai/kernels/magic/queues/manager.py`

全局执行结果队列：

```python
execution_result_queue = defaultdict(FasterQueue)  # { uuid: Queue }
```

**特点**：
- 按 uuid 隔离队列，不同执行互不干扰
- 使用 `spawn` 启动方式（跨平台兼容）
- 支持同步和异步两种获取方式

### 6.4 前端 SSE 客户端

**Hook 实现**：`mage_ai/frontend/utils/server/events/useEventStreams.ts`

**核心功能**：

| 功能 | 实现 |
|-----|------|
| 自动连接 | useEffect 中自动建立 EventSource |
| 自动重连 | 指数退避，最多 10 次 |
| 消息缓存 | events 数组、messages 数组 |
| 发送消息 | 通过 HTTP POST /code_executions 发送 |
| 状态追踪 | CONNECTING / OPEN / RECONNECTING / CLOSED |

**重连策略**（指数退避）：

```
第 1 次重连 → 等待 1 秒
第 2 次重连 → 等待 2 秒
第 3 次重连 → 等待 3 秒
...
第 10 次后 → 停止
```

### 6.5 消息类型

**EventStreamTypeEnum**（`mage_ai/frontend/interfaces/EventStreamType.ts`）：

| 类型 | 说明 |
|-----|------|
| `execution` | 执行结果 |
| `execution_status` | 执行状态 |
| `task` | 任务信息 |
| `task_status` | 任务状态 |

**ExecutionStatusEnum**：
- `RUNNING` — 运行中
- `SUCCESS` — 成功
- `FAILURE` — 失败
- `ERROR` — 错误

### 6.6 SSE 的状态同步边界

| 边界项 | 说明 |
|-------|------|
| **数据缓冲** | 队列中待消费的消息，消费后即丢弃 |
| **消息 ID** | 无 `Last-Event-ID` 支持，重连后无法续传 |
| **事件类型** | 所有消息都是默认事件类型，前端无法按类型订阅 |
| **重试间隔** | 未设置 retry 字段，依赖浏览器默认值 |
| **队列溢出** | 无队列长度限制，消费不及时可能内存溢出 |
| **多路复用** | 一个 uuid 一个连接，多流时连接数多 |
| **结束标记** | 流无明确结束信号，前端不知道何时完成 |
| **活跃连接管理** | active_connections 已定义但未使用 |
| **空载优化** | consecutive_sleep_count 已定义但未使用 |

---

## 七、四者界限与适用范围

### 7.1 功能界限矩阵

| 功能场景 | 前端轮询 | 数据请求缓存 | WebSocket | SSE |
|---------|---------|-------------|-----------|-----|
| MonitorStats 统计图表 | ✅ 主力 | ✅ 辅助缓存 | ❌ 不使用 | ❌ 不使用 |
| PipelineRun 历史列表 | ✅ 主力 | ✅ 辅助缓存 | ❌ 不使用 | ❌ 不使用 |
| BlockRun 历史列表 | ✅ 主力 | ✅ 辅助缓存 | ❌ 不使用 | ❌ 不使用 |
| 交互式块执行 | ❌ 不使用 | ❌ 不相关 | ✅ 主力 | ❌ 不使用 |
| 管道执行（UI 触发） | ❌ 不使用 | ❌ 不相关 | ✅ 主力 | ❌ 不使用 |
| 代码执行输出流 | ❌ 不使用 | ❌ 不相关 | ❌ 不使用 | ✅ 主力 |
| 系统状态监控 | ✅ 辅助 | ✅ 辅助缓存 | ❌ 不使用 | ❌ 不使用 |
| 终端仿真 | ❌ 不使用 | ❌ 不相关 | ✅ 主力 | ❌ 不使用 |
| 运行中状态刷新 | ✅ 轮询辅助 | ✅ 缓存展示 | ✅ 实时推送 | ❌ 不使用 |
| 数据去重合并 | ❌ 不适用 | ✅ 主力 | ❌ 不适用 | ❌ 不适用 |

### 7.2 数据类型与实时性要求

| 数据类型 | 实时性要求 | 同步机制 | 延迟 |
|---------|-----------|---------|------|
| 历史运行统计 | 低（分钟级） | HTTP 轮询 + 缓存 | 60s / 无轮询 |
| 运行详情状态 | 中（秒级） | HTTP 轮询 + 缓存 | 3-5s |
| 块执行输出 | 高（毫秒级） | WebSocket | 近实时 |
| 管道执行状态 | 高（毫秒级） | WebSocket | 近实时 |
| 代码执行结果流 | 高（毫秒级） | SSE | ~0.1s |
| 系统状态 | 低（秒级） | HTTP 延迟轮询 | 3-7s 延迟 + 轮询 |
| 重复查询缓存 | 瞬时 | SWR 缓存 | 0ms（本地读取） |

### 7.3 状态所有权划分

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           数据所有权模型                                │
├───────────────┬──────────────┬──────────────┬───────────────────────────┤
│ 机制          │ 状态所有者   │ 数据持久化   │ 前端缓存策略              │
├───────────────┼──────────────┼──────────────┼───────────────────────────┤
│ 前端轮询      │ 服务端数据库 │ 持久化存储   │ SWR 全局内存缓存          │
│ (HTTP + SWR)  │ (单一真相源) │              │ URL 为 key               │
│               │              │              │ 轮询刷新                  │
├───────────────┼──────────────┼──────────────┼───────────────────────────┤
│ 数据请求缓存  │ 前端内存     │ 不持久化     │ 直接读写缓存              │
│ (SWR Cache)   │ (缓存副本)   │              │ 随页面刷新重置            │
├───────────────┼──────────────┼──────────────┼───────────────────────────┤
│ WebSocket     │ 服务端内存   │ 不持久化     │ 组件本地 state            │
│               │ (运行时状态) │              │ 消息追加模式              │
│               │              │              │ 连接重建后丢失            │
├───────────────┼──────────────┼──────────────┼───────────────────────────┤
│ SSE           │ 服务端队列   │ 不持久化     │ 组件本地 state            │
│               │ (缓冲队列)   │              │ 事件追加模式              │
│               │              │              │ 消费后即丢弃              │
└───────────────┴──────────────┴──────────────┴───────────────────────────┘
```

### 7.4 适用范围总结

| 机制 | 适用场景 | 不适用场景 |
|-----|---------|-----------|
| **前端轮询** | 历史数据统计、低频变化数据、列表展示 | 实时交互式操作、高频状态更新 |
| **数据请求缓存** | 重复查询、页面切换快速展示、降低服务端压力 | 需要强一致的实时数据、首次访问加速 |
| **WebSocket** | 交互式代码执行、管道控制、终端仿真、双向通信 | 历史数据查询、大量客户端广播 |
| **SSE** | 单向数据流、执行结果输出、日志流推送 | 需要双向通信、需要消息确认 |

### 7.5 互补与协作

**管道运行详情页的多机制协作**：

在管道运行详情页（`pipelines/[pipeline]/runs/[run]`），存在三种机制的协作：

1. **SWR 缓存** — 提供快速的初始数据展示（如果之前请求过）
2. **HTTP 轮询** — 定期刷新运行状态和日志
3. **WebSocket** — 如果用户在编辑页面触发执行，实时状态通过 WebSocket 推送

**关键协作点**：
- WebSocket 连接断开时，HTTP 轮询作为保底机制
- 管道执行结束（idle 状态）时，前端触发 `fetchPipeline()` 刷新完整数据
- 轮询间隔动态调整：运行中 3s，空闲时停止
- SWR 缓存提供毫秒级的初始渲染，轮询在后台更新

---

## 八、未覆盖的功能与能力边界

### 8.1 前端轮询未覆盖能力

| 功能点 | 当前状态 | 影响 |
|-------|---------|------|
| **自适应轮询** | ❌ 固定间隔，不根据数据变化频率调整 | 数据变化慢时浪费请求，变化快时不及时 |
| **后台同步** | ❌ 标签页不可见时仍在轮询（部分浏览器会节流） | 资源浪费 / 后台数据不更新 |
| **批量查询** | ❌ 每个资源独立请求，无批量合并 | 请求数多，首屏加载慢 |
| **增量更新** | ❌ 每次全量拉取 | 数据量大时浪费带宽 |
| **断点续传** | ❌ 无 ETag / Last-Modified 支持 | 重复传输相同数据 |
| **离线队列** | ❌ 离线时请求直接失败 | 离线状态下无数据 |
| **数据订阅** | ❌ 轮询即订阅，无法取消 | 后台页面持续消耗资源 |

### 8.2 数据请求缓存未覆盖能力

| 功能点 | 当前状态 | 影响 |
|-------|---------|------|
| **缓存持久化** | ❌ 仅内存缓存 | 刷新页面后全部丢失 |
| **缓存 TTL** | ❌ 无过期时间配置 | 数据可能长时间不刷新 |
| **缓存淘汰** | ❌ 无 LRU/LFU 策略 | 长期使用内存膨胀 |
| **乐观更新** | ❌ 未使用 mutate 的乐观更新能力 | 用户操作后有延迟感 |
| **缓存失效联动** | ❌ 相关资源修改后不联动失效 | 列表和详情可能不一致 |
| **部分缓存** | ❌ 整个响应作为缓存单元 | 无法利用部分缓存 |
| **缓存预热** | ❌ 无预加载机制 | 首次访问慢 |

### 8.3 WebSocket 未覆盖能力

| 功能点 | 当前状态 | 影响 |
|-------|---------|------|
| **消息确认机制** | ❌ 发送后不验证客户端是否收到 | 网络不稳定时可能丢消息 |
| **消息持久化** | ❌ 连接期间的消息不保存 | 重连后丢失历史 |
| **连接数限制** | ❌ 无最大连接数控制 | 大量连接可能耗尽资源 |
| **消息限流** | ❌ 无限速 | 高频输出可能阻塞 |
| **房间/频道机制** | ❌ 所有客户端都收到所有消息 | 隐私和性能问题 |
| **心跳检测** | ❌ 无 ping/pong 机制 | 死连接可能长时间不察觉 |
| **消息顺序保证** | ❌ 依赖 TCP 但无应用层序号 | 极端情况可能乱序 |
| **单用户多连接** | ❌ 无法识别同一用户的多个连接 | 无法定向推送 |
| **执行状态恢复** | ❌ 刷新页面后执行状态丢失 | 用户体验中断 |
| **消息压缩** | ❌ 未使用压缩 | 大数据量时带宽浪费 |
| **二进制支持** | ❌ 仅文本消息 | 无法传输二进制数据 |
| **重连状态同步** | ❌ 重连后不补历史消息 | 可能错过关键状态 |

### 8.4 SSE 未覆盖能力

| 功能点 | 当前状态 | 影响 |
|-------|---------|------|
| **消息 ID** | ❌ 无 Last-Event-ID 支持 | 重连后无法续传 |
| **事件类型** | ❌ 所有消息都是默认事件类型 | 前端无法按类型订阅 |
| **重试间隔** | ❌ 未设置 retry 字段 | 依赖浏览器默认值 |
| **队列溢出保护** | ❌ 无队列长度限制 | 消费不及时可能内存溢出 |
| **多路复用** | ❌ 一个 uuid 一个连接 | 多流时连接数多 |
| **结束标记** | ❌ 流无明确结束信号 | 前端不知道何时完成 |
| **活跃连接管理** | ❌ active_connections 定义但未使用 | 无法管理和监控连接 |
| **空载优化** | ❌ consecutive_sleep_count 定义但未使用 | 空转时 CPU 浪费 |
| **双向通信** | ❌ SSE 本身仅服务端推送，发送走 HTTP | 发送延迟高，需单独鉴权 |
| **二进制数据** | ❌ SSE 仅支持文本 | 无法传输二进制流 |

### 8.5 状态同步整体边界

| 维度 | 当前状态 | 理想状态 |
|-----|---------|---------|
| **一致性模型** | 最终一致（轮询间隔内可能不一致） | 可选择强一致 / 最终一致 |
| **离线支持** | 无任何离线能力 | 离线缓存 + 上线同步 |
| **状态冲突** | 无冲突检测，最后写入生效 | 冲突检测 + 解决策略 |
| **部分失败** | 单点失败（如 WebSocket 断了就没实时数据） | 多通道冗余 + 优雅降级 |
| **可观测性** | 无内部监控 | 自监控（连接数、消息量、延迟） |
| **扩展性** | 单实例内存状态 | 支持集群 / 分布式部署 |
| **统一状态层** | 各机制独立管理状态 | 统一状态管理层，多通道同步 |

### 8.6 MonitorStats 业务统计未覆盖能力

| 功能点 | 当前状态 | 影响 |
|-------|---------|------|
| **实时性** | ❌ 监控页无自动刷新 | 用户看到的可能是过时数据 |
| **数据分页** | ❌ 一次性返回所有数据 | 大量块 / 调度时性能差 |
| **服务端缓存** | ❌ 每次请求都查数据库 | 数据库压力大 |
| **告警配置** | ❌ 无法设置阈值告警 | 纯展示，无主动通知 |
| **数据导出** | ❌ 无法导出 CSV / Excel | 数据只能在 UI 查看 |
| **聚合粒度** | ❌ 仅支持按日聚合 | 无法看小时级 / 周级趋势 |
| **对比分析** | ❌ 无同比环比 | 无法判断趋势好坏 |
| **错误处理** | ❌ 错误仅 print | 排障困难 |
| **权限控制** | ❌ 所有用户看到相同数据 | 多租户下数据隔离问题 |

---

## 九、关键代码索引

### 9.1 前端轮询 & 数据请求缓存

| 文件路径（相对仓库根） | 说明 |
|----------------------|------|
| `mage_ai/frontend/api/utils/use.ts` | SWR 封装核心，useDetail/useList 等 Hook |
| `mage_ai/frontend/api/index.ts` | API 资源自动生成器，RESOURCES_PAIRS_ARRAY 配置 |
| `mage_ai/frontend/api/utils/useDelayFetch.ts` | 延迟拉取 Hook，默认 delay = 3000ms |
| `mage_ai/frontend/pages/overview/index.tsx` | 概览页，60s 手动轮询 monitor_stats |
| `mage_ai/frontend/pages/pipelines/[pipeline]/monitors/index.tsx` | 管道运行监控页，无自动刷新 |
| `mage_ai/frontend/pages/pipelines/[pipeline]/monitors/block-runs.tsx` | 块运行监控页，切换调度手动刷新 |
| `mage_ai/frontend/pages/pipelines/[pipeline]/monitors/block-runtime.tsx` | 块运行时间监控页，useEffect 触发刷新 |
| `mage_ai/frontend/utils/models/status/useStatus.ts` | 系统状态 Hook，默认 delay = 7000ms |
| `mage_ai/frontend/pages/_app.tsx` | 全局入口，useStatus delay = 3000ms |

### 9.2 WebSocket

| 文件路径（相对仓库根） | 说明 |
|----------------------|------|
| `mage_ai/server/websocket_server.py` | WebSocket 服务端核心，连接管理和消息分发 |
| `mage_ai/server/websockets/models.py` | 消息模型、客户端模型、错误模型 |
| `mage_ai/server/websockets/utils.py` | 消息处理工具，认证、过滤、脱敏 |
| `mage_ai/server/websockets/constants.py` | 常量定义（Channel、ExecutionState、MessageType） |
| `mage_ai/server/execution_manager.py` | 管道执行管理，多进程执行和状态同步 |
| `mage_ai/frontend/api/utils/url.ts` | WebSocket / SSE URL 构建函数 |
| `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | 管道编辑页，WebSocket 主使用场景 |

### 9.3 SSE 事件流

| 文件路径（相对仓库根） | 说明 |
|----------------------|------|
| `mage_ai/server/events/stream.py` | SSE 服务端，EventStreamHandler 实现 |
| `mage_ai/kernels/magic/queues/manager.py` | 执行结果队列管理，全局队列单例 |
| `mage_ai/shared/queues.py` | 队列抽象层，faster_fifo / multiprocessing 适配 |
| `mage_ai/frontend/utils/server/events/useEventStreams.ts` | 前端 SSE Hook，连接管理和重连逻辑 |
| `mage_ai/frontend/interfaces/EventStreamType.ts` | SSE 类型定义，事件类型和状态枚举 |

### 9.4 业务统计后端

| 文件路径（相对仓库根） | 说明 |
|----------------------|------|
| `mage_ai/orchestration/monitor/monitor_stats.py` | MonitorStats 核心类，4 种统计类型实现 |
| `mage_ai/api/resources/MonitorStatResource.py` | API 资源层，REST 接口封装 |
| `mage_ai/system/memory/manager.py` | 内存监控管理器，上下文管理器模式 |
| `mage_ai/api/resources/StatusResource.py` | 系统状态 API 资源 |

---

## 十、总结

### 10.1 设计亮点

1. **分层清晰** — 四种机制各司其职：历史统计用 HTTP 轮询、缓存加速用 SWR、交互式执行用 WebSocket、结果流用 SSE
2. **SWR 全局缓存** — 避免重复请求，提升用户体验，相同 URL 的组件共享数据
3. **自动重连** — WebSocket 和 SSE 都有重连机制，增强可靠性
4. **安全考虑** — 敏感数据过滤、OAuth 认证、权限校验层层把关
5. **跨平台兼容** — 队列层抽象（faster_fifo / multiprocessing）、数据库层抽象（PostgreSQL / SQLite）
6. **动态调速** — 运行详情页根据状态动态调整轮询频率，节省资源
7. **延迟加载** — useDelayFetch 机制避免页面初始化时的请求风暴

### 10.2 主要不足

1. **监控页实时性弱** — 三个监控统计页面均无自动刷新，与"监控"的定位存在落差
2. **状态孤岛** — HTTP 轮询、WebSocket、SSE 各管各的状态，没有统一的状态管理层
3. **无持久化消息** — WebSocket 和 SSE 的消息都是瞬时的，刷新即丢失
4. **可扩展性差** — 所有状态都在单进程内存中，无法水平扩展
5. **缺少自监控** — 监控系统本身没有自监控能力（连接数、消息量、延迟等指标）
6. **缓存策略简陋** — SWR 缓存无 TTL、无淘汰、无持久化，长期使用可能内存膨胀
7. **WebSocket 广播风暴** — 所有客户端收到所有消息，无房间/频道机制

### 10.3 潜在风险

1. **WebSocket 广播风暴** — 所有客户端收到所有消息，客户端数 × 消息数 = O(n²) 复杂度
2. **SSE 队列内存溢出** — 无队列长度限制，消费不及时可能导致内存泄漏
3. **数据库压力** — MonitorStats 每次都查全量数据，高并发下可能成为瓶颈
4. **状态丢失** — WebSocket 断开期间的执行状态无法恢复，用户体验中断
5. **缓存不一致** — SWR 缓存和 WebSocket 实时数据之间可能出现短暂不一致
6. **轮询浪费** — 固定频率轮询在数据无变化时造成不必要的服务器压力
7. **死连接** — 无心跳检测，WebSocket 死连接可能长时间不被察觉

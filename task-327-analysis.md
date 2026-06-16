# 监控与 Stats 模块代码分析

## 一、架构总览

Mage AI 的监控体系分为两大分支：**业务运行统计**（MonitorStats）和 **系统资源监控**（MemoryManager）。整体采用三层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                     前端展示层 (Frontend)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Pipeline Runs│  │ Block Runs   │  │ Block Runtime    │  │
│  │ 监控页面     │  │ 监控页面     │  │ 监控页面         │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │            │
│         └─────────────────┼────────────────────┘            │
│                           │                                 │
│                    SWR (React Query)                        │
│                    轮询 + 缓存                               │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP API
┌───────────────────────────▼─────────────────────────────────┐
│                      API 层 (API Layer)                       │
│  ┌──────────────────────┐   ┌────────────────────────────┐  │
│  │ MonitorStatResource  │   │ StatusResource             │  │
│  │ 业务统计资源          │   │ 系统状态资源               │  │
│  └──────────┬───────────┘   └──────────────┬─────────────┘  │
│             │                              │                │
└─────────────┼──────────────────────────────┼────────────────┘
              │                              │
┌─────────────▼──────────────────────────────▼────────────────┐
│                   数据层 (Data Layer)                          │
│  ┌──────────────────────┐   ┌────────────────────────────┐  │
│  │ MonitorStats (业务)  │   │ MemoryManager (系统)       │  │
│  │ - PipelineRun 统计   │   │ - 内存使用监控             │  │
│  │ - BlockRun 统计      │   │ - 日志文件持久化           │  │
│  └──────────┬───────────┘   └──────────────┬─────────────┘  │
│             │                              │                │
│  ┌──────────▼───────────┐   ┌──────────────▼─────────────┐  │
│  │  数据库 (SQLAlchemy) │   │  文件系统 (.log)           │  │
│  │  PipelineRun 表      │   │  按日期/小时分区           │  │
│  │  BlockRun 表         │   │  Polars 解析聚合           │  │
│  └──────────────────────┘   └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

除此之外，实时执行状态通过 **WebSocket** 和 **SSE (Server-Sent Events)** 推送到前端。

---

## 二、核心步骤分析

### 2.1 业务统计（MonitorStats）

**核心文件**：[monitor_stats.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py)

#### 2.1.1 入口与分发

`MonitorStats.get_stats()` 是统一入口，根据 `stats_type` 分发到四个具体方法：

| 统计类型 | 枚举值 | 说明 |
|---------|--------|------|
| 管道运行次数 | `PIPELINE_RUN_COUNT` | 按日期分组统计各状态管道运行数量 |
| 管道运行时间 | `PIPELINE_RUN_TIME` | 按日期统计管道平均运行时长 |
| 块运行次数 | `BLOCK_RUN_COUNT` | 按日期分组统计各状态块运行数量 |
| 块运行时间 | `BLOCK_RUN_TIME` | 按日期统计块平均运行时长 |

**调用流程**：

```
前端请求 → MonitorStatResource.member()
                │
                ▼
        MonitorStats.get_stats(stats_type, ...)
                │
     ┌──────────┼──────────┬──────────┐
     ▼          ▼          ▼          ▼
  管道次数   管道时长    块次数     块时长
```

#### 2.1.2 管道运行次数统计（get_pipeline_run_count）

核心步骤：
1. **数据库查询**：从 `PipelineRun` 表 JOIN `PipelineSchedule` 表，获取创建时间、调度ID、管道UUID、状态等字段
2. **日期格式化**：使用数据库函数 `format_datetime()` 将 `created_at` 格式化为 `YYYY-MM-DD` 字符串（SQL 层面完成，提升性能）
3. **管道元数据映射**：遍历查询结果中的唯一 `pipeline_uuid`，从文件系统加载 Pipeline 对象（用于获取 pipeline type）
4. **按调度分组聚合**：
   - 外层按 `pipeline_schedule_id` 分组
   - 内层按日期字符串分组
   - 最内层按运行状态计数（或按管道类型再嵌套一层）

**关键代码结构**（[monitor_stats.py#L93-L190](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L93-L190)）：

```python
# 查询使用 SQL 层聚合 + Python 层二次聚合的混合模式
query = PipelineRun.select(...).join(PipelineSchedule, ...)
query = self.__filter(query, ...)  # 应用时间范围、管道等过滤条件
pipeline_runs = query.all()

# Python 层进行多维聚合
for p in pipeline_runs:
    # 按 schedule_id → 日期 → 状态 三级嵌套
    data[created_at_formatted][p.status] += 1
```

#### 2.1.3 管道运行时间统计（get_pipeline_run_time）

核心步骤：
1. 查询已完成的管道运行（`completed_at != NULL` 且 `created_at != NULL`）
2. 按日期分组后计算平均运行时间
3. 过滤异常数据：`completed_at > created_at` 才参与计算

**平均值计算函数**（[monitor_stats.py#L232-L237](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L232-L237)）：

```python
def __mean_runtime(pipeline_runs):
    runtime_list = [(p.completed_at - p.created_at).total_seconds()
                    for p in pipeline_runs if p.completed_at > p.created_at]
    if len(runtime_list) == 0:
        return 0
    return sum(runtime_list) / len(runtime_list)
```

#### 2.1.4 块运行次数统计（get_block_run_count）

核心步骤：
1. 使用 SQL `GROUP BY` 进行数据库层面的预聚合（`block_uuid` + `status` + `ds_created_at`）
2. 使用 `func.count()` 直接在 SQL 层完成计数
3. Python 层使用 `reduce()` 将数据重组为嵌套字典结构

**SQL 层聚合优化**（[monitor_stats.py#L254-L294](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L254-L294)）：

```python
# 关键：使用 SQL GROUP BY 减少数据传输量
block_runs = (
    query.
    group_by('name', BlockRun.status, 'ds_created_at', ...).
    all()
)
```

#### 2.1.5 块运行时间统计（get_block_run_time）

核心步骤：
1. 过滤已完成的 BlockRun
2. 先按 `block_uuid` 分组，再按日期分组
3. 对每组应用统计函数（平均运行时间）

### 2.2 系统内存监控（MemoryManager）

**核心文件**：
- [manager.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/manager.py) - 内存管理器
- [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/utils.py) - 工具函数
- [wrappers.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/wrappers.py) - 函数包装器

#### 2.2.1 上下文管理器模式

`MemoryManager` 采用上下文管理器模式（`with` 语句），支持同步和异步两种方式：

```python
# 同步使用
with MemoryManager(scope_uuid, process_uuid) as mm:
    # 执行需要监控的代码
    ...

# 异步使用
async with MemoryManager(scope_uuid, process_uuid) as mm:
    # 执行需要监控的代码
    ...
```

#### 2.2.2 生命周期

**启动阶段**（`__enter__` / `__aenter__`）：
1. 创建日志目录（按日期/小时分区）
2. 启动后台监控线程，定期采样内存
3. 写入 START 标记和初始内存值

**运行阶段**：
- 监控线程按 `poll_interval` 间隔采样（默认 `SYSTEM_LOGS_POLL_INTERVAL`）
- 每次采样写入一条日志记录

**停止阶段**（`__exit__` / `__aexit__`）：
1. 写入当前内存值
2. 写入 END 标记
3. 停止监控线程

#### 2.2.3 日志文件结构

日志文件路径模式（[manager.py#L62-L86](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/manager.py#L62-L86)）：

```
{variables_dir}/system/metrics/
    /pipelines/{pipeline_uuid}/{block_uuid}
        /{date}/{hour}
            /memory
                /{process_uuid}.log
```

日志行格式：
```
[timestamp][log_type] message key1=value1 key2=value2
```

#### 2.2.4 内存监控线程

[monitor_memory_usage](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/utils.py#L602-L621) 函数创建监控线程：

- 使用 `threading.Event` 作为停止信号
- 使用 `memory_profiler.memory_usage()` 获取当前内存
- 回调函数负责将内存值写入日志

### 2.3 实时状态推送（WebSocket）

**核心文件**：[websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/websocket_server.py)

#### 2.3.1 连接建立

1. 客户端连接到 `/websocket/` 端点
2. 服务端将客户端加入 `WebSocketServer.clients` 集合
3. 等待客户端发送消息

#### 2.3.2 消息处理流程

收到消息后：
1. **认证**：检查 API key + token，验证 OAuth 权限
2. **权限校验**：检查用户对管道的操作权限
3. **分发执行**：
   - `cancel_pipeline`：取消管道执行
   - `execute_pipeline`：执行整个管道
   - 单个块执行：通过 Jupyter kernel 执行代码

#### 2.3.3 状态广播

`WebSocketServer.send_message()` 是消息广播入口（[websocket_server.py#L312-L399](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/websocket_server.py#L312-L399)）：

1. **消息过滤**：跳过无效消息（无数据、无错误、无状态、无类型）
2. **敏感数据过滤**：根据 `HIDE_ENV_VAR_VALUES` 配置过滤环境变量
3. **错误格式化**：简化堆栈跟踪，移除内部 Mage 方法调用
4. **元数据注入**：注入 `block_uuid`、`pipeline_uuid`、`block_type` 等
5. **广播**：遍历所有连接的客户端，写入消息

---

## 三、状态同步机制

### 3.1 HTTP 轮询模式（业务统计）

业务统计数据采用 **HTTP 轮询 + 客户端缓存** 的模式。

#### 3.1.1 API 资源层

[MonitorStatResource](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/api/resources/MonitorStatResource.py) 是 REST API 入口：

- 只有 `member` 方法（详情接口），没有列表接口
- 以 `stats_type` 作为资源 ID（`pk`）
- 支持查询参数：`pipeline_uuid`、`start_time`、`end_time`、`pipeline_schedule_id`、`group_by_pipeline_type`

#### 3.1.2 前端数据获取

前端使用 **SWR (stale-while-revalidate)** 模式获取数据：

```typescript
// 概览页面每 60 秒刷新一次（[overview/index.tsx#L93-L96]）
const SHARED_FETCH_OPTIONS = {
  refreshInterval: 60000,      // 60秒轮询
  revalidateOnFocus: false,    // 聚焦时不刷新
};
```

在管道监控页面，数据通过 `api.monitor_stats.detail()` 获取，由 SWR 管理缓存。

#### 3.1.3 数据转换

前端获取原始统计数据后，需要进行适配转换：

1. **日期范围补全**：使用 `getDateRange()` 生成连续日期数组，将统计数据按日期填充
2. **格式转换**：将嵌套字典转换为图表组件需要的数组格式
3. **聚合计算**：概览页使用 `getAllPipelineRunDataGrouped()` 汇总所有管道数据

### 3.2 WebSocket 推送模式（执行状态）

管道和块的实时执行状态通过 WebSocket 推送。

#### 3.2.1 执行状态映射

服务端维护 `running_executions_mapping` 字典（[websocket_server.py#L168](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/websocket_server.py#L168)）：

```python
WebSocketServer.running_executions_mapping = {
    msg_id: {
        block_type: '...',
        block_uuid: '...',
        replicated_block: '...',
        pipeline_uuid: '...',
    }
}
```

当 kernel 返回执行结果时，通过 `msg_id` 查找对应的块元数据，注入到消息中再广播给客户端。

#### 3.2.2 管道执行状态

对于管道执行，状态通过 **多进程队列** 传递：

1. 主进程创建 `multiprocessing.Queue`
2. 子进程（`run_pipeline`）执行管道，通过队列发送状态消息
3. 主进程启动 `check_for_messages` 协程，轮询队列并通过 WebSocket 广播

### 3.3 SSE 事件流模式

**核心文件**：[stream.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/events/stream.py)

`EventStreamHandler` 提供 SSE (Server-Sent Events) 流式接口：

- 端点：`/event-streams/{uuid}`
- 从执行结果队列（`FasterQueue`）非阻塞获取消息
- 每 0.1 秒轮询一次队列
- 以 `text/event-stream` 格式推送

### 3.4 文件系统持久化模式（系统监控）

系统监控数据通过文件系统持久化，支持离线分析。

#### 3.4.1 日志解析

[presenters.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/presenters.py) 提供日志解析能力：

1. 遍历目录下所有 `.log` 文件
2. 逐行解析日志格式，提取 `timestamp`、`log_type`、`message`、`metadata` 等字段
3. 使用 **Polars DataFrame** 进行高效聚合分析

#### 3.4.2 单例控制器

[MemoryManagerController](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/shared/singletons/memory.py) 是全局单例，管理所有监控线程：

```python
# 注册监控事件
add_event_monitor(key, stop_event, monitor_thread)

# 停止单个事件
stop_event(key)

# 停止所有事件
stop_all_events()
```

---

## 四、边界条件分析

### 4.1 错误处理

#### 4.1.1 Pipeline 加载失败

在 `get_pipeline_run_count` 中，加载 Pipeline 元数据时有容错处理（[monitor_stats.py#L138-L147](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L138-L147)）：

```python
try:
    pipeline = Pipeline.get(uuid, ...)
    pipelines_mapping[pipeline.uuid] = pipeline
except Exception as err:
    print(f'[ERROR] MonitorStats.get_pipeline_run_count: {err}.')
```

**影响**：如果管道文件被删除或损坏，`group_by_pipeline_type` 功能会跳过该管道的数据，但不影响整体统计结果。

#### 4.1.2 内存监控异常

内存监控的退出阶段有 `try-except-finally` 保护（[manager.py#L125-L132](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/manager.py#L125-L132)）：

```python
try:
    with open(self.log_path, 'a') as f:
        f.write(...)  # 写入结束标记
except Exception as err:
    print(f'[MemoryManager] Error writing memory usage logs: {err}')
finally:
    self.stop()  # 确保线程被停止
```

**设计原则**：即使日志写入失败，也必须确保监控线程被正确停止，避免资源泄漏。

#### 4.1.3 WebSocket 消息过滤

无效消息在广播前被过滤（[websocket_server.py#L314-L332](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/websocket_server.py#L314-L332)）：

- 空消息（无 data、无 error、无 execution_state、无 type）被丢弃
- Jupyter widget 相关消息（如 `FloatProgress`）被过滤

#### 4.1.4 运行时间异常值

计算平均运行时间时，过滤掉 `completed_at <= created_at` 的异常数据（[monitor_stats.py#L233-L234](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L233-L234)）：

```python
runtime_list = [(p.completed_at - p.created_at).total_seconds()
                for p in pipeline_runs if p.completed_at > p.created_at]
```

### 4.2 数据边界

#### 4.2.1 时间范围默认值

时间参数为 `None` 时的默认行为（[monitor_stats.py#L44-L51](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L44-L51)）：

- `end_time` 默认：当前时间（`datetime.utcnow()`）
- `start_time` 默认：结束时间往前 30 天（`end_time - timedelta(days=30)`）

#### 4.2.2 空数据处理

- 无运行数据时，`__mean_runtime` 返回 0 而非报错
- 无 pipeline_schedule_id 时，使用 `NO_PIPELINE_SCHEDULE_ID` 作为占位符

#### 4.2.3 数据库兼容性

日期格式化函数根据数据库类型动态调整（[functions.py#L16-L40](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/db/functions.py#L16-L40)）：

- PostgreSQL：使用 `to_char(column, 'YYYY-MM-DD')`
- SQLite：使用 `strftime('%Y-%m-%d', column)`

### 4.3 性能考虑

#### 4.3.1 查询优化痕迹

代码中保留了性能计时的 print 语句（[monitor_stats.py#L132](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py#L132)），显示了优化过程：

```
# Query: 11.1094 → 2.7102 (after optimization)
# Mapping: 0.2497
# Loop: 2.4017 → 1.0568 (optimized)
```

**主要优化手段**：
1. **SQL 层预聚合**：`get_block_run_count` 使用 `GROUP BY` 在数据库层完成计数，减少数据传输
2. **日期格式化下推**：使用数据库函数格式化日期，避免 Python 层 `strftime` 开销
3. **集合去重**：使用 `set([p.pipeline_uuid for p in pipeline_runs])` 减少 Pipeline 加载次数

#### 4.3.2 内存监控开销

- 使用独立线程监控，不阻塞主线程
- 轮询间隔可配置（`SYSTEM_LOGS_POLL_INTERVAL`）
- `MEMORY_MANAGER_V2` 开关控制是否启用完整功能

#### 4.3.3 前端性能

- 使用 SWR 缓存减少重复请求
- `useMemo` 缓存计算结果，避免重复数据转换
- 图表组件按需渲染

### 4.4 并发与资源管理

#### 4.4.1 单例与线程安全

`MemoryManagerController` 使用单例模式（通过 `@singleton` 装饰器），内部通过事件字典管理所有监控线程。

#### 4.4.2 WebSocket 多客户端

`WebSocketServer.clients` 是类级别的集合，所有连接的客户端共享。消息广播时遍历所有客户端。

**潜在风险**：大量客户端连接时，广播可能成为瓶颈。

#### 4.4.3 资源清理

- 内存监控：`__exit__` 中确保调用 `stop()` 停止线程
- WebSocket：`on_close` 时从 `clients` 集合移除
- 管道执行：取消时通过 `cancel_pipeline_execution` 终止子进程

---

## 五、关键模块索引

### 5.1 业务统计相关

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| MonitorStats 类 | [monitor_stats.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/monitor/monitor_stats.py) | 核心统计逻辑 |
| MonitorStatResource | [MonitorStatResource.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/api/resources/MonitorStatResource.py) | API 资源层 |
| 前端类型定义 | [MonitorStatsType.ts](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/frontend/interfaces/MonitorStatsType.ts) | TypeScript 类型 |
| 管道运行监控页 | [monitors/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/frontend/pages/pipelines/[pipeline]/monitors/index.tsx) | 管道运行统计页面 |
| 块运行监控页 | [block-runs.tsx](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/frontend/pages/pipelines/[pipeline]/monitors/block-runs.tsx) | 块运行统计页面 |
| 块运行时间监控页 | [block-runtime.tsx](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/frontend/pages/pipelines/[pipeline]/monitors/block-runtime.tsx) | 块运行时长页面 |
| Monitor 组件 | [Monitor/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/frontend/components/Monitor/index.tsx) | 监控布局组件 |

### 5.2 系统监控相关

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| MemoryManager | [manager.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/manager.py) | 内存管理器 |
| 内存工具函数 | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/utils.py) | 内存采集、估算工具 |
| 日志解析器 | [presenters.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/presenters.py) | 日志转 Polars DataFrame |
| 内存追踪包装器 | [wrappers.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/system/memory/wrappers.py) | 函数级内存追踪 |
| 内存控制器单例 | [memory.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/shared/singletons/memory.py) | 全局监控线程管理 |

### 5.3 实时通信相关

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| WebSocket 服务 | [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/websocket_server.py) | WebSocket 消息处理 |
| SSE 事件流 | [stream.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/server/events/stream.py) | SSE 事件流推送 |
| StatusResource | [StatusResource.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/api/resources/StatusResource.py) | 系统状态 API |

### 5.4 数据库模型

| 模型 | 文件路径 | 说明 |
|------|---------|------|
| PipelineRun | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/db/models/schedules.py) | 管道运行记录 |
| BlockRun | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/db/models/schedules.py) | 块运行记录 |
| PipelineSchedule | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/327-mage-ai/mage_ai/orchestration/db/models/schedules.py) | 管道调度配置 |

---

## 六、实现边界总结

### 6.1 业务统计（MonitorStats）的边界

**已实现的功能**：
- ✅ 管道运行次数统计（按状态、按调度、按管道类型）
- ✅ 管道运行时间统计（平均时长）
- ✅ 块运行次数统计（按状态、按块）
- ✅ 块运行时间统计（平均时长）
- ✅ 时间范围过滤
- ✅ 管道 UUID 过滤
- ✅ 调度 ID 过滤
- ✅ 多数据库支持（PostgreSQL / SQLite）

**未覆盖的边界**：
- ❌ 没有数据分页机制（大数据量时可能有性能问题）
- ❌ 没有缓存层（每次请求都查数据库）
- ❌ 错误仅 print 不记录日志
- ❌ 不支持按小时/周/月等粒度聚合
- ❌ 没有统计指标的告警阈值配置

### 6.2 系统监控（MemoryManager）的边界

**已实现的功能**：
- ✅ 进程级内存使用监控
- ✅ 同步/异步上下文管理器
- ✅ 日志文件持久化（按日期/小时分区）
- ✅ 内存使用估算（多种文件类型、对象类型）
- ✅ 日志解析与 Polars 聚合
- ✅ 全局线程管理

**未覆盖的边界**：
- ❌ 仅监控内存，不监控 CPU、磁盘 IO 等
- ❌ 没有实时告警机制
- ❌ 日志文件没有自动清理策略
- ❌ 缺少分布式环境下的聚合能力

### 6.3 实时通信的边界

**已实现的功能**：
- ✅ WebSocket 双向通信
- ✅ SSE 事件流
- ✅ 执行状态实时推送
- ✅ 多客户端广播
- ✅ OAuth 认证

**未覆盖的边界**：
- ❌ 没有消息确认/重传机制
- ❌ 没有连接数限制
- ❌ 没有消息持久化
- ❌ 不支持消息历史回放

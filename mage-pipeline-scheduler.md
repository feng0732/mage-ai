# Mage AI Pipeline 执行调度分析

## 1. 核心流程概述

Mage AI 的 Pipeline 调度系统采用**定时轮询 + 事件驱动**的混合架构，以 `schedule_all()` 为主调度入口，通过多层级状态机协同完成任务触发、依赖判断、执行排队与结果记录。

### 1.1 调度主循环

调度器由 [scheduler_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/server/scheduler_manager.py) 启动独立进程运行：

1. `SchedulerManager.start_scheduler()` 创建独立进程，调用 `run_scheduler()`
2. `run_scheduler()` 启动 `JobManager` 和 `LoopTimeTrigger`
3. [LoopTimeTrigger](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/triggers/loop_time_trigger.py) 以默认 10 秒（`SCHEDULER_TRIGGER_INTERVAL`）间隔循环执行：
   - `schedule_all()` — 调度所有 Pipeline
   - `check_sla()` — 检查 SLA 违约

### 1.2 调度决策流程

[schedule_all()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1656-L1954) 函数执行以下关键步骤：

```
同步 YAML 触发器配置到 DB
    ↓
获取所有 ACTIVE 状态的 PipelineSchedule
    ↓
按 Pipeline 分组，逐个处理
    ├─ 获取分布式锁 pipeline_schedule_{id}
    ├─ PipelineSchedule.should_schedule() 判断是否需调度
    ├─ 检查并发限制并创建 PipelineRun（INITIAL 状态）
    ├─ 按并发配额启动 PipelineRun（→ RUNNING）
    └─ 对超限任务执行 WAIT/SKIP 策略
    ↓
遍历所有 RUNNING 状态的 PipelineRun
    └─ PipelineScheduler(r).schedule() 调度 Block
```

---

## 2. 协作模块详解

### 2.1 任务触发模块

#### 2.1.1 PipelineSchedule 调度决策

[PipelineSchedule.should_schedule()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/db/models/schedules.py#L537-L679) 根据 ScheduleType 执行不同判断逻辑：

| ScheduleInterval | 调度条件 |
|---|---|
| `ONCE` | 无历史运行记录；或流式 Pipeline 运行数 < executor_count |
| `ALWAYS_ON` | 无运行记录；或上次运行非 INITIAL/RUNNING |
| 时间型（HOURLY/DAILY 等） | 当前执行周期尚无同 execution_date 的运行记录；启用 landing_time 时需考虑历史平均运行时间提前调度 |
| Cron 表达式 | 由 croniter 计算最近执行时间 |

**关键特性**：
- `landing_time_enabled`：基于 `runtime_history()` 历史平均耗时 + 标准差，提前启动以保证在目标时间前完成
- `create_initial_pipeline_run`：控制 schedule 启用时是否立即补跑当前周期
- `skip_if_previous_running`：上一周期未完成则跳过当前周期（标记为 CANCELLED）

#### 2.1.2 事件触发

[schedule_with_event()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L2067-L2107) 通过 `EventMatcher.match(event)` 匹配事件模式，命中即创建 PipelineRun。

### 2.2 依赖判断模块

#### 2.2.1 Block 可执行性判断

[PipelineRun.executable_block_runs()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/db/models/schedules.py#L978-L1221) 是依赖判断核心，处理以下场景：

1. **普通 Block**：检查 `block.upstream_blocks` 是否均为 COMPLETED 状态
2. **Dynamic Block 子节点**：调用 `check_all_dynamic_upstreams_completed()` 验证所有动态上游
3. **动态块索引（metrics.dynamic_block_index）**：检查指定动态上游列表是否完成
4. **数据集成子块**：
   - `original` 块需等待所有非 controller 子块完成
   - controller 子块且非并行时，需等待其配置的上游块
5. **allow_blocks_to_fail 开关**：True 时将 FAILED/UPSTREAM_FAILED 也视为"已完成"，False 时仅 COMPLETED 有效

#### 2.2.2 失败状态传播

[PipelineRun.update_block_run_statuses()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1223-L1301) 递归传播失败状态：
- 上游 FAILED/UPSTREAM_FAILED → 当前块标记为 `UPSTREAM_FAILED`
- 上游 CONDITION_FAILED → 当前块标记为 `CONDITION_FAILED`

### 2.3 运行状态管理

#### 2.3.1 状态枚举

**PipelineRun 状态**（[PipelineRunStatus](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/db/models/schedules.py#L769-L774)）：
| 状态 | 含义 |
|---|---|
| INITIAL | 已创建，待调度 |
| RUNNING | 执行中 |
| COMPLETED | 全部 Block 成功完成 |
| FAILED | 有 Block 失败且 allow_blocks_to_fail=False |
| CANCELLED | 用户取消或被跳过 |

**BlockRun 状态**（[BlockRunStatus](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1664-L1672)）：
| 状态 | 含义 |
|---|---|
| INITIAL | 待调度 |
| QUEUED | 已入执行队列 |
| RUNNING | 执行中 |
| COMPLETED | 成功完成 |
| FAILED | 执行失败 |
| CANCELLED | 被取消 |
| UPSTREAM_FAILED | 上游失败导致跳过 |
| CONDITION_FAILED | 条件判断失败导致跳过 |

#### 2.3.2 心跳与健康检查

[PipelineScheduler.__run_heartbeat()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L906-L933) 在每次 schedule() 时执行：
- 记录 CPU load15 / CPU 核数 / 内存使用率
- 若内存使用率 ≥ 95%（`MEMORY_USAGE_MAXIMUM`）且使用本地 Python 执行器，则调用 `memory_usage_failure()` 停止 Pipeline 并发送通知

#### 2.3.3 超时检查

| 检查对象 | 方法 | 配置源 | 超时动作 |
|---|---|---|---|
| PipelineRun | [__check_pipeline_run_timeout()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L529-L555) | PipelineSchedule.settings.timeout | 标记 FAILED（或配置的 timeout_status），取消所有 Block |
| BlockRun | [__check_block_run_timeout()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L558-L599) | Block.timeout | 标记 FAILED，kill 对应 Job |

#### 2.3.4 崩溃恢复

[PipelineScheduler.__fetch_crashed_block_runs()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L871-L904) 检测处于 RUNNING/QUEUED 状态但对应 Job 已不存在的 BlockRun，将其重置为 INITIAL 以便重跑。

### 2.4 执行队列模块

#### 2.4.1 JobManager 抽象层

[JobManager](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/job_manager.py) 定义统一接口，底层通过 `QueueFactory` 选择实现。

**JobType 分类**：
| JobType | 用途 | ID 格式 |
|---|---|---|
| BLOCK_RUN | 单个 Block 执行 | `block_run_{block_run_id}` |
| PIPELINE_RUN | 整 Pipeline 单进程执行或集成流顺序执行 | `pipeline_run_{pipeline_run_id}` |
| INTEGRATION_STREAM | 数据集成并行流 | `integration_stream_{pipeline_run_id}_{stream}` |
| GENERIC_JOB | 通用任务（如取消回调） | `generic_job_{id}` |

#### 2.4.2 ProcessQueue 实现

[ProcessQueue](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/queue/process_queue.py) 基于 Python multiprocessing：

- **Worker Pool**：`poll_job_and_execute()` 主循环维护 worker 进程池，默认大小为 CPU 核数（`queue_config.concurrency`）
- **跨实例同步**：配置 Redis 时通过 Redis 记录 `job_id → client_id` 与 `client_id` 心跳（TTL 300s），防止多副本重复执行
- **Job 生命周期**：
  ```
  enqueue → job_dict[QUEUED] → Worker.run → job_dict[pid] → COMPLETED
                                                         ↘ (kill) → CANCELLED
  ```
- **分布式 kill**：通过 Redis key `kill_job_{job_id}` 通知其他实例终止任务

#### 2.4.3 调度策略

[PipelineScheduler.schedule()](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L213-L338) 根据 Pipeline 类型选择不同调度模式：

| Pipeline 类型 / 配置 | 调度方式 |
|---|---|
| STREAMING | `__schedule_pipeline()` 整体入队 |
| INTEGRATION | `__schedule_integration_streams()` 按 stream 拆分，支持并行/顺序流 |
| run_pipeline_in_one_process=True | `__schedule_pipeline()` 单进程执行整 Pipeline |
| 其他（标准 Batch） | `__schedule_blocks()` 按 Block 粒度入队 |

---

## 3. 核心机制深度分析

### 3.1 并发管理

[ConcurrencyConfig](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/concurrency.py) 通过三层配额控制并发：

```
pipeline_run_limit_all_triggers （Pipeline 全局）
    ↓
pipeline_run_limit （单 Schedule/Trigger）
    ↓
block_run_limit （单 PipelineRun 内）
```

配置来源优先级：`Pipeline.concurrency_config` > 环境变量 `CONCURRENCY_CONFIG_*`。

**超限策略**（`on_pipeline_run_limit_reached`）：
- `WAIT`（默认）：保留 INITIAL 状态，等待下一轮调度
- `SKIP`：直接标记 CANCELLED 并记录日志

### 3.2 失败重试策略

#### 3.2.1 Block 级重试

[shared/retry.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/shared/retry.py) 提供装饰器：

```python
@retry(retries=2, delay=5, max_delay=60, exponential_backoff=True)
```

配置来源（优先级从高到低）：
1. `run_block()` 传入的 `retry_config` 参数
2. `Block.retry_config`（Pipeline YAML 中 per-block 配置）
3. `RepoConfig.retry_config`（全局）

**数据集成 Pipeline** 的 Block 强制 `retries=0`，不执行重试。

#### 3.2.2 Pipeline 级重试

通过 API/UI 手动触发 `retry_pipeline_run()`，创建新的 PipelineRun 复用原 execution_date、variables、event_variables 等。

### 3.3 状态保持与分布式一致性

#### 3.3.1 分布式锁

[DistributedLock](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/utils/distributed_lock.py) 基于 Redis SETNX 实现：
- 锁键：`LOCK_KEY_{key}`
- 默认超时：30 秒（`REDIS_LOCK_DEFAULT_TIMEOUT`）
- 无 Redis 时退化为**无锁直接通过**（单实例场景）

**关键锁点**：
| 锁键 | 获取位置 | 目的 |
|---|---|---|
| `pipeline_run_{id}` | PipelineScheduler.start / schedule | 防止同一 PipelineRun 并发启动/调度 |
| `pipeline_schedule_{id}` | schedule_all() 内层 | 防止同一 Schedule 并发创建运行 |

#### 3.3.2 数据库状态持久化

所有状态（PipelineRun、BlockRun、PipelineSchedule）均存储于关系型数据库（PostgreSQL/SQLite），调度器每轮循环 `db_connection.session.expire_all()` 强制刷新会话缓存以保证读取最新数据。

### 3.4 调度限制与保护机制

| 限制项 | 阈值 | 触发动作 |
|---|---|---|
| 内存使用率 | ≥ 95% | 停止 Pipeline，发送告警通知 |
| Pipeline 超时 | settings.timeout（秒） | 标记 FAILED / timeout_status |
| Block 超时 | block.timeout（秒） | 标记 FAILED 并 kill Job |
| SLA | schedule.sla（秒） | 标记 passed_sla=True，发送 SLA 告警 |
| skip_if_previous_running | 布尔 | 上轮未完则跳过本轮 |
| 重复运行检测 | execution_date 唯一 | 同 Schedule 同 execution_date 不重复创建 |

---

## 4. 潜在风险分析

### 4.1 竞态条件

1. **多调度器副本并发**：`schedule_all()` 注释明确指出当前并发限制检查与 API/事件触发之间可能存在竞态，尽管使用分布式锁，但在锁获取前的配额读取仍有窗口期
2. **INITIAL 状态 PipelineRun 无锁保护**：`pipeline_run_{id}` 锁仅在 start() 和 schedule() 中获取，而 INITIAL → RUNNING 的转换依赖 `should_schedule()` 与并发计数之间的非原子操作
3. **Redis 不可用时的退化**：DistributedLock 在 Redis 不可用时直接返回 True（无锁），ProcessQueue.has_job() 也仅依赖本地 job_dict，多实例场景下会重复执行

### 4.2 可靠性风险

1. **崩溃恢复不完整**：`__fetch_crashed_block_runs()` 仅重置 BlockRun 状态，但不清理可能已在队列中的脏 Job；若进程异常退出，Redis 中 job_id 键需等待 300s 心跳 TTL 过期才能清除
2. **状态机缺少 SHUTDOWN 处理**：调度器进程被 SIGTERM 时未对 RUNNING 状态的 PipelineRun/BlockRun 做持久化标记，重启后依赖崩溃恢复逻辑
3. **回调执行异步化**：`on_pipeline_run_cancelled` 通过 GenericJob 异步执行，取消动作与回调之间存在时间差，且 GenericJob 自身失败无重试

### 4.3 性能与扩展性

1. **全局串行调度**：`schedule_all()` 在单线程内按 Pipeline 顺序处理，Pipeline 数量大时单轮耗时增长，无法分片
2. **N+1 查询问题**：`executable_block_runs()` 内对每个 Block 多次调用 `pipeline.get_block()`，Block 数量大时产生大量 DB/IO 查询
3. **Worker Pool 固定大小**：ProcessQueue 的 worker 数在启动时确定，无法按任务类型（CPU/IO 密集）动态伸缩
4. **每轮全量扫描**：调度循环每轮扫描所有 ACTIVE Schedule 和所有 RUNNING PipelineRun，随规模增长线性变慢

### 4.4 可观测性

1. **调度循环自身无指标**：`schedule_all()` 无执行耗时、处理 Pipeline/Block 数等内部指标上报，仅依赖日志
2. **重试元数据不完整**：Block 的实际重试次数、每次重试的错误未结构化存入 metrics，仅通过日志输出
3. **分布式锁获取失败静默**：锁获取失败直接 continue，无告警或统计，调度遗漏难以发现

---

## 5. 后续调查方向

### 5.1 高优先级

1. **分布式一致性验证**：在多副本部署场景下，构造高并发触发（API + 时间调度同时），验证是否会生成重复 PipelineRun 或 Block 被重复执行
2. **Redis 故障演练**：模拟 Redis 不可用/网络分区，观察调度行为是否符合预期（单副本正常、多副本可能重复执行），并评估对业务的影响
3. **崩溃恢复路径测试**：强制 kill 调度进程与 worker 进程，验证 PipelineRun 和 BlockRun 是否能在重启后正确恢复并最终达成终态

### 5.2 中优先级

4. **大规模调度压测**：构造 1000+ PipelineSchedule、单 Pipeline 含 100+ Block 的场景，测量 `schedule_all()` 单轮耗时、数据库连接池压力、内存占用
5. **重试策略实测**：验证 Block 级指数退避在 DB 连接超时、外部 API 抖动等真实故障下的有效性，并统计端到端成功率
6. **并发限制有效性**：验证 `pipeline_run_limit_all_triggers` 在跨 Schedule 场景是否正确生效，检查计数是否包含 INITIAL 状态

### 5.3 低优先级 / 优化方向

7. **调度分片方案**：评估将 PipelineSchedule 按 hash 分片到多个调度实例（或引入调度主节点）以提升水平扩展能力
8. **状态机持久化增强**：为 PipelineRun/BlockRun 增加 `last_heartbeat_at` 字段，由 worker 定期刷新，调度器可更精准地判断僵尸任务
9. **指标埋点**：为调度主循环、锁获取、队列深度、重试次数等关键路径添加 Prometheus 指标
10. **取消流程原子性**：评估将 kill job + 更新 BlockRun 状态 + 触发回调封装为更原子的流程，减少中间状态窗口

---

## 6. 关键代码文件索引

| 模块 | 文件路径 | 核心内容 |
|---|---|---|
| 调度主入口 | [pipeline_scheduler_original.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py) | schedule_all(), PipelineScheduler 类, run_block/run_pipeline |
| 数据模型 | [db/models/schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/db/models/schedules.py) | PipelineSchedule, PipelineRun, BlockRun, GenericJob, EventMatcher |
| 调度进程管理 | [server/scheduler_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/server/scheduler_manager.py) | SchedulerManager, LoopTimeTrigger 启动 |
| 触发器实现 | [orchestration/triggers/](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/triggers/) | TimeTrigger, LoopTimeTrigger, EventTrigger |
| Job 管理层 | [orchestration/job_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/job_manager.py) | JobManager, JobType 枚举 |
| 执行队列实现 | [queue/process_queue.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/queue/process_queue.py) | ProcessQueue, Worker, poll_job_and_execute |
| 并发配置 | [orchestration/concurrency.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/concurrency.py) | ConcurrencyConfig, OnLimitReached |
| 分布式锁 | [utils/distributed_lock.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/utils/distributed_lock.py) | DistributedLock（Redis 锁） |
| 重试工具 | [shared/retry.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/shared/retry.py) | @retry 装饰器（指数退避） |
| 进程执行管理 | [execution_process_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/execution_process_manager.py) | ExecutionProcessManager（本地进程跟踪） |
| 服务端配置 | [settings/server.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/settings/server.py) | SCHEDULER_TRIGGER_INTERVAL, CONCURRENCY_CONFIG_* |
| 状态检查辅助 | [run_status_checker.py](file:///d:/fz/0601/solo-dogfeeding/code/308-mage-ai/mage_ai/orchestration/run_status_checker.py) | Sensor 式上游状态查询 |

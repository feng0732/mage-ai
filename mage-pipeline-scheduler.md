# Mage AI Pipeline 执行调度分析

## 1. 核心流程概述

Mage AI 的 Pipeline 调度系统采用**定时轮询 + 事件驱动**的混合架构，以 `schedule_all()` 为主调度入口，通过多层级状态机协同完成任务触发、依赖判断、执行排队与结果记录。

### 1.1 调度主循环

调度器由 `mage_ai/server/scheduler_manager.py` 启动独立进程运行：

1. `SchedulerManager.start_scheduler()` 创建独立进程，调用 `run_scheduler()`
2. `run_scheduler()` 启动 `JobManager` 和 `LoopTimeTrigger`
3. `mage_ai/orchestration/triggers/loop_time_trigger.py` 以默认 10 秒（`SCHEDULER_TRIGGER_INTERVAL`）间隔循环执行：
   - `schedule_all()` — 调度所有 Pipeline
   - `check_sla()` — 检查 SLA 违约

### 1.2 调度决策流程

`mage_ai/orchestration/pipeline_scheduler_original.py` `schedule_all()`（L1656–L1954）执行以下关键步骤：

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

`mage_ai/orchestration/db/models/schedules.py` `PipelineSchedule.should_schedule()`（L537–L679）根据 ScheduleType 执行不同判断逻辑：

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

`mage_ai/orchestration/pipeline_scheduler_original.py` `schedule_with_event()`（L2067–L2107）通过 `EventMatcher.match(event)` 匹配事件模式，命中即创建 PipelineRun。

### 2.2 依赖判断模块

#### 2.2.1 Block 可执行性判断

`mage_ai/orchestration/db/models/schedules.py` `PipelineRun.executable_block_runs()`（L978–L1221）是依赖判断核心，处理以下场景：

1. **普通 Block**：通过 `all_upstream_blocks_completed(completed_block_uuids)` 检查上游是否全部 COMPLETED（`completed_block_uuids` 仅含 COMPLETED 状态的 block_uuid）
2. **Dynamic Block 子节点**：调用 `check_all_dynamic_upstreams_completed()` 验证所有动态上游（仅认 COMPLETED，不受 `allow_blocks_to_fail` 影响）
3. **动态块索引（metrics.dynamic_block_index）**：根据 `allow_blocks_to_fail` 选择检查 `completed_block_uuids`（仅 COMPLETED）或 `finished_block_uuids`（含 COMPLETED/FAILED/UPSTREAM_FAILED）
4. **数据集成子块**：
   - `original` 块需等待所有非 controller 子块完成
   - controller 子块且非并行时，需等待其配置的上游块
5. **`allow_blocks_to_fail` 的作用点**：不在可执行判断层面改变普通 Block 的逻辑，而是通过入口终止控制和失败传播间接影响（详见 4.5 节三层影响分析）

#### 2.2.2 失败状态传播

`mage_ai/orchestration/db/models/schedules.py` `PipelineRun.update_block_run_statuses()`（L1223–L1301）递归传播失败状态：
- 上游 FAILED/UPSTREAM_FAILED → 当前块标记为 `UPSTREAM_FAILED`
- 上游 CONDITION_FAILED → 当前块标记为 `CONDITION_FAILED`

### 2.3 运行状态管理

#### 2.3.1 状态枚举

**PipelineRun 状态**（`mage_ai/orchestration/db/models/schedules.py` L769–L774）：
| 状态 | 含义 |
|---|---|
| INITIAL | 已创建，待调度 |
| RUNNING | 执行中 |
| COMPLETED | 全部 Block 成功完成 |
| FAILED | 有 Block 失败（`allow_blocks_to_fail=False` 时立即终止；`True` 时仅当 `all_blocks_completed` 后仍有失败 Block 才标记） |
| CANCELLED | 用户取消或被跳过 |

**BlockRun 状态**（`mage_ai/orchestration/db/models/schedules.py` L1664–L1672）：
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

`mage_ai/orchestration/pipeline_scheduler_original.py` `PipelineScheduler.__run_heartbeat()`（L906–L933）在每次 schedule() 时执行：
- 记录 CPU load15 / CPU 核数 / 内存使用率
- 若内存使用率 ≥ 95%（`MEMORY_USAGE_MAXIMUM`）且使用本地 Python 执行器，则调用 `memory_usage_failure()` 停止 Pipeline 并发送通知

#### 2.3.3 超时检查

| 检查对象 | 方法 | 配置源 | 超时动作 |
|---|---|---|---|
| PipelineRun | `__check_pipeline_run_timeout()`（L529–L555） | PipelineSchedule.settings.timeout | 标记 FAILED（或配置的 timeout_status），取消所有 Block |
| BlockRun | `__check_block_run_timeout()`（L558–L599） | Block.timeout | 标记 FAILED，kill 对应 Job |

#### 2.3.4 崩溃恢复

`mage_ai/orchestration/pipeline_scheduler_original.py` `PipelineScheduler.__fetch_crashed_block_runs()`（L871–L904）检测处于 RUNNING/QUEUED 状态但对应 Job 已不存在的 BlockRun，将其重置为 INITIAL 以便重跑。

### 2.4 执行队列模块

#### 2.4.1 JobManager 抽象层

`mage_ai/orchestration/job_manager.py` 定义统一接口，底层通过 `QueueFactory` 选择实现。

**JobType 分类**：
| JobType | 用途 | ID 格式 |
|---|---|---|
| BLOCK_RUN | 单个 Block 执行 | `block_run_{block_run_id}` |
| PIPELINE_RUN | 整 Pipeline 单进程执行或集成流顺序执行 | `pipeline_run_{pipeline_run_id}` |
| INTEGRATION_STREAM | 数据集成并行流 | `integration_stream_{pipeline_run_id}_{stream}` |
| GENERIC_JOB | 通用任务（如取消回调） | `generic_job_{id}` |

#### 2.4.2 ProcessQueue 实现

`mage_ai/orchestration/queue/process_queue.py` 基于 Python multiprocessing：

- **Worker Pool**：`poll_job_and_execute()` 主循环维护 worker 进程池，默认大小为 CPU 核数（`queue_config.concurrency`）
- **跨实例同步**：配置 Redis 时通过 Redis 记录 `job_id → client_id` 与 `client_id` 心跳（TTL 300s），防止多副本重复执行
- **Job 生命周期**：
  ```
  enqueue → job_dict[QUEUED] → Worker.run → job_dict[pid] → COMPLETED
                                                         ↘ (kill) → CANCELLED
  ```
- **分布式 kill**：通过 Redis key `kill_job_{job_id}` 通知其他实例终止任务

#### 2.4.3 调度策略

`mage_ai/orchestration/pipeline_scheduler_original.py` `PipelineScheduler.schedule()`（L213–L338）根据 Pipeline 类型选择不同调度模式：

| Pipeline 类型 / 配置 | 调度方式 |
|---|---|
| STREAMING | `__schedule_pipeline()` 整体入队 |
| INTEGRATION | `__schedule_integration_streams()` 按 stream 拆分，支持并行/顺序流 |
| run_pipeline_in_one_process=True | `__schedule_pipeline()` 单进程执行整 Pipeline |
| 其他（标准 Batch） | `__schedule_blocks()` 按 Block 粒度入队 |

---

## 3. 核心机制深度分析

### 3.1 并发管理

`mage_ai/orchestration/concurrency.py` `ConcurrencyConfig` 通过三层配额控制并发：

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

`mage_ai/shared/retry.py` 提供装饰器：

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

`mage_ai/orchestration/utils/distributed_lock.py` `DistributedLock` 基于 Redis SETNX 实现：
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

## 4. 结果记录闭环

本节阐述 Block 执行完毕后，从数据输出到最终影响后续调度决策的完整闭环路径。该闭环是理解调度系统可靠性的关键——任何环节的断裂都可能导致下游 Block 误判上游状态、数据丢失或调度死锁。

### 4.1 闭环全景

```
BlockExecutor.execute()
  ├─ 成功路径                                         ├─ 失败路径
  │   ↓                                               │   ↓
  │   _execute() 内 Block.execute_sync() 执行         │   __execute_with_retry() 重试耗尽
  │   ↓                                               │   ↓
  │   execute_sync() 内 store_variables() 持久化输出   │   on_failure(block_uuid, error) 回调
  │   ↓                                               │   ↓
  │   on_complete(block_uuid) 回调                     │   on_block_failure(block_uuid, error)
  │   ↓                                               │   ↓
  │   BlockRun → COMPLETED                            │   BlockRun → FAILED
  │   ↓                                               │   ↓
  │   PipelineScheduler.schedule() 重新进入            │   PipelineScheduler.schedule() 重新进入
  │   ↓                                               │   ↓
  │   executable_block_runs() 读取最新状态             │   update_block_run_statuses() 传播失败
  │   ↓                                               │   ↓
  │   下游 Block 获得调度资格                          │   下游 Block → UPSTREAM_FAILED
  │                                                    │   ↓
  │                                                    │   ├─ allow_blocks_to_fail=True  → 继续调度
  │                                                    │   └─ allow_blocks_to_fail=False → PipelineRun.FAILED
  └────────────────────────────────────────────────────┘
```

**关键时序**：变量写入（步骤 ②）和状态写回（步骤 ④）是**串行但有间隔**的两个操作，中间的失败会走异常路径，它们的完成与否共同决定 BlockRun 的最终状态。

### 4.2 块输出持久化

Block 的数据输出通过两条路径持久化，下游 Block 依赖这些输出来执行自身逻辑：

**路径 1：磁盘变量存储**（默认路径，`cache_block_output_in_memory=False`）

`mage_ai/data_preparation/models/block/__init__.py` `Block.store_variables()`（L3776–L3861）将执行结果写入变量管理系统：
- `variable_manager.add_variable()` 将输出数据（DataFrame、对象等）以 `pipeline_uuid/block_uuid/variable_uuid` 为键存储到磁盘
- 写入前先通过 `__store_variables_prepare()` 清洗、序列化变量映射
- Dynamic Block 子节点执行前会调用 `delete_variable_objects_for_dynamic_child()` 清除旧变量
- 数据集成 Pipeline 的源/目标 Block 由于返回值包含不可序列化的进程对象，`store_variables=False`，其输出由专门的流式处理路径处理

**路径 2：内存缓存**（`cache_block_output_in_memory=True`）

`mage_ai/data_preparation/executors/pipeline_executor.py` `PipelineExecutor.__run_blocks()`（L94–L171）在同进程模式下：
- 维护 `block_run_outputs_cache: Dict[str, List]`，key 为 block_uuid
- `asyncio.gather()` 并行执行所有可执行 Block 后，将结果直接缓存：`block_run_outputs_cache[block_run.block_uuid] = block_run_outputs[idx].get('output', [])`
- 后续 Block 通过 `input_from_output` 参数直接从缓存读取上游输出，无需磁盘 I/O
- **注意**：此模式下 `store_variables` 被强制设为 `False`（`block_executor.py` L830–L833），输出仅存于内存，进程退出即丢失

### 4.3 变量写入失败后的状态走向

`store_variables()` 的调用位于 `Block.execute_sync()` 的 `try` 块内（L1621–L1640）。失败后的状态走向取决于异常类型和回调链：

#### 4.3.1 异常在 execute_sync 内传播

```
execute_sync()
  try:
    output = self.execute_block(...)          ← 业务逻辑执行成功
    block_output = self.post_process_output(output)
    variable_mapping = dict(zip(variable_keys, block_output))
    if store_variables:
      self._store_variables_in_block_function(variable_mapping)  ← 此处可能失败
  except ValueError as e:
    if str(e) == 'Circular reference detected':  ← 特殊处理循环引用
      raise ValueError('Please provide dataframe or json serializable data as output.')
    raise e                                       ← 其他 ValueError 继续抛出
  except Exception as err:                        ← 其他异常
    if update_status:
      self.status = BlockStatus.FAILED            ← Block 内存状态标记为 FAILED
    raise err                                     ← 向上传播
```

**关键路径**：
1. **Circular reference** — 抛出更明确的 ValueError，最终被 `BlockExecutor.execute()` 的 `except Exception` 捕获 → 调用 `on_failure` → `on_block_failure()` → BlockRun 标记 FAILED
2. **序列化/IO 错误** — `store_variables()` 内 `variable_manager.add_variable()` 写磁盘失败时，异常向上传播至 `execute_sync()` 的 `except Exception` → Block 内存状态 FAILED → 异常继续抛出到 `BlockExecutor.execute()` → `on_failure` → BlockRun 标记 **FAILED**（**不是 COMPLETED**）
3. **业务逻辑成功 + 变量写入失败**：BlockRun 状态为 FAILED，与业务逻辑本身的失败不可区分

#### 4.3.2 变量写入失败对下游的影响

| 场景 | BlockRun 状态 | 变量是否存在 | 下游 Block 行为 |
|---|---|---|---|
| 业务成功 + 变量写入成功 | COMPLETED | 是 | 正常读取上游输出 |
| 业务成功 + 变量写入失败 | FAILED | 否/部分 | 不被 `executable_block_runs()` 选中（FAILED 不在 `completed_block_uuids` 中；若 `allow_blocks_to_fail=True`，则 FAILED 在 `finished_block_uuids` 中，但普通 Block 不使用 `finished_block_uuids`，且下游已被传播为 UPSTREAM_FAILED） |
| 业务失败 + 变量未写入 | FAILED | 否 | 同上 |

**校正之前的错误认知**：此前认为"store_variables 失败但 BlockRun 标记 COMPLETED"，这是不正确的。实际上 `store_variables()` 失败的异常会传播到 `BlockExecutor.execute()` 的 `except` 分支，触发 `on_failure` 回调，BlockRun 最终标记为 **FAILED**。

### 4.4 运行结果与状态写回

Block 执行完毕后，状态写回分两种场景：

#### 4.4.1 标准 Batch Pipeline（按 Block 粒度调度）

调度器调用 `mage_ai/orchestration/pipeline_scheduler_original.py` `run_block()`（L1230–L1333）时，根据 `schedule_after_complete` 参数决定回调方式：

| 回调 | 何时使用 | 行为 |
|---|---|---|
| `on_block_complete` | `schedule_after_complete=True`（默认） | 更新 BlockRun → COMPLETED，**然后调用 `schedule()` 继续调度下游** |
| `on_block_complete_without_schedule` | `schedule_after_complete=False`（数据集成流内） | 仅更新 BlockRun → COMPLETED，**不触发后续调度** |

`on_block_complete()` 的写回逻辑（L377–L411）：

```python
@retry(retries=2, delay=5)
def update_status(metrics=metrics):
    metrics_prev = block_run.metrics or {}
    if metrics:
        metrics_prev.update(metrics)
    block_run.update(
        status=BlockRun.BlockRunStatus.COMPLETED,
        completed_at=datetime.now(tz=pytz.UTC),
        metrics=metrics_prev,
    )
```

关键细节：
- metrics 使用**合并更新**（`metrics_prev.update(metrics)`），后续调度轮次读到的是累积的完整 metrics
- `completed_at` 仅在此处设置，`started_at` 在 `run_block()` 入口处设置
- DB 更新失败时 `@retry(retries=2, delay=5)` 最多重试 2 次，间隔 5 秒
- 写回成功后检查 `pipeline_run.status` 是否仍为 RUNNING，是则调用 `self.schedule()` 进入下一轮调度
- **BlockRun 已由 `run_block()` 入口处设为 RUNNING**，此处从 RUNNING → COMPLETED

#### 4.4.2 单进程 Pipeline（`run_pipeline_in_one_process`）

`PipelineExecutor.__run_blocks()` 自行管理状态写回：
- Block 开始前：`block_run.update(status=RUNNING, started_at=now)`（L121–L126）
- Block 成功后：`BlockExecutor.execute()` 内部调用 `on_complete`（即 `on_block_complete_without_schedule`），仅写回 COMPLETED 状态
- Block 失败后：`execute_block()` 抛出异常 → `asyncio.gather()` 收集异常 → **整个 `__run_blocks()` 因未捕获异常而终止**
- 所有 Block 完成后，由 `PipelineScheduler.schedule()` 检测 `all_blocks_completed()` 并标记 PipelineRun 终态

#### 4.4.3 单进程执行异常对终态的影响

单进程模式下的异常传播路径与标准 Batch 模式有本质区别：

**标准 Batch 模式**：Block 在独立进程中执行，异常被 `BlockExecutor.execute()` 捕获 → `on_failure` 回调 → BlockRun FAILED → `on_block_failure()` → 调度器 `schedule()` 继续运行，判断 PipelineRun 终态

**单进程模式**（`PipelineExecutor.__run_blocks()`）：
1. Block 失败时 `BlockExecutor.execute()` 内部捕获异常 → 调用 `on_failure`（即 `on_block_failure` 的回调引用，L681–L685）→ BlockRun 标记 FAILED
2. **但** `execute_block()` 的 `raise error`（L150）将异常重新抛出
3. `asyncio.gather(*block_run_tasks)` 收集到异常后，`__run_blocks()` 函数整体退出（L167 之后无更多逻辑）
4. `PipelineExecutor.execute()` 的 `_execute_task()` 退出 → `run_pipeline()` 函数退出
5. **PipelineRun 状态停留在 RUNNING**——没有任何代码将 PipelineRun 标记为 FAILED
6. 依赖下一轮 `schedule_all()` → `PipelineScheduler.schedule()` 检测到 `any_blocks_failed() and not allow_blocks_to_fail` → 将 PipelineRun 标记为 FAILED

**关键区别**：
- 标准模式：失败立即传播到 PipelineRun（同轮 `schedule()` 内完成）
- 单进程模式：失败传播延迟到下一轮 `schedule_all()`（最多 10 秒间隔）
- 单进程模式下的 `allow_blocks_to_fail=True` 场景：`__run_blocks()` 的 while 循环中 `all_blocks_completed(allow_blocks_to_fail)` 为 False 时，`any_blocks_failed()` 为 True 但 `allow_blocks_to_fail` 为 True → 循环继续，但 `executable_block_runs()` 返回空（因为失败 Block 的下游被标记 UPSTREAM_FAILED）→ 循环退出 → 同样依赖下一轮 `schedule()` 处理

#### 4.4.4 条件失败写回

`mage_ai/data_preparation/executors/block_executor.py` `_execute_conditional()` 返回 False 时（L397–L467）：
- 非 DI 场景：`__update_block_run_status(CONDITION_FAILED)` 直接写回单个 BlockRun
- DI 场景：递归调用 `__update_condition_failed()` 将当前块及所有下游块都标记为 CONDITION_FAILED
- CONDITION_FAILED 状态的 BlockRun 不再被 `executable_block_runs()` 返回，但也**不算作 FAILED**，因此 `any_blocks_failed()` 返回 False，Pipeline 仍可正常完成

### 4.5 失败传播与 allow_blocks_to_fail 的三层影响

`allow_blocks_to_fail` 配置从三个层面影响调度行为：入口终止、失败传播、可执行判断。三者相互关联但作用点不同，必须分开理解。

#### 4.5.1 第一层：入口终止（PipelineScheduler.schedule）

`PipelineScheduler.schedule()`（L213–L338）的入口判断：

```
if all_blocks_completed(self.allow_blocks_to_fail):
    if any_blocks_failed():
        PipelineRun → FAILED
    else:
        PipelineRun → COMPLETED
elif any_blocks_failed() and not self.allow_blocks_to_fail:
    PipelineRun → FAILED          ← 立即终止
else:
    继续调度
```

- `all_blocks_completed(include_failed_blocks)`（L1522–L1532）：当 `include_failed_blocks=False` 时，仅 COMPLETED + CONDITION_FAILED 算完成；当 `True` 时，FAILED + UPSTREAM_FAILED 也算完成
- `any_blocks_failed()`（L1519–L1520）：检查是否有 `status == FAILED` 的 BlockRun

| allow_blocks_to_fail | any_blocks_failed()=True 时的行为 |
|---|---|
| **False** | 在入口处立即将 PipelineRun 标记 FAILED，取消所有 Block 和 Job |
| **True** | 不在入口终止，继续调度后续 Block |

#### 4.5.2 第二层：失败传播（update_block_run_statuses）

`PipelineRun.update_block_run_statuses()`（`mage_ai/orchestration/db/models/schedules.py` L1223–L1301）在 `__schedule_blocks()` 入口调用，**不受 `allow_blocks_to_fail` 影响**，始终执行：

```
Block A FAILED
    ↓ update_block_run_statuses()
Block B（A 的下游，INITIAL 状态）→ UPSTREAM_FAILED
    ↓ 递归（因 BlockRun 集合变化，not_updated 列表缩短）
Block C（B 的下游，INITIAL 状态）→ UPSTREAM_FAILED
```

传播逻辑：
1. 收集所有 FAILED/UPSTREAM_FAILED 块的 `block_uuid` → `failed_block_uuids`
2. 收集所有 CONDITION_FAILED 块的 `block_uuid` → `condition_failed_block_uuids`
3. 对每个 INITIAL 状态的 BlockRun，检查其 `upstream_block_uuids` 是否与上述集合有交集
4. 有交集则更新为对应状态，无变化则加入 `not_updated_block_runs`
5. 如果本轮有更新，递归调用自身直到无新变化

**关键**：无论 `allow_blocks_to_fail` 如何配置，失败传播始终发生。区别在于：
- `allow_blocks_to_fail=False`：传播结果无实际意义——PipelineRun 已在入口处标记 FAILED，不会再调度
- `allow_blocks_to_fail=True`：传播后的 UPSTREAM_FAILED BlockRun 不再是 INITIAL，不会进入 `executable_block_runs()` 的候选列表

#### 4.5.3 第三层：可执行判断（executable_block_runs）

`PipelineRun.executable_block_runs()`（L978–L1221）判断哪些 BlockRun 可以被调度。**此处的关键前提**是正确理解 `completed_block_uuids` 和 `finished_block_uuids` 的含义：

**`completed_block_runs` 属性**（L849–L850）**仅包含 `status == COMPLETED` 的 BlockRun**：

```python
@property
def completed_block_runs(self) -> List['BlockRun']:
    return [b for b in self.block_runs if b.status == BlockRun.BlockRunStatus.COMPLETED]
```

**`_build_block_uuids()` 辅助函数**（L1003–L1021）过滤 COMPLETED + UPSTREAM_FAILED + FAILED：

```python
def _build_block_uuids(block_runs: List[BlockRun]) -> List[str]:
    arr = set()
    for block_run in block_runs:
        if block_run.status in [
            BlockRun.BlockRunStatus.COMPLETED,
            BlockRun.BlockRunStatus.UPSTREAM_FAILED,
            BlockRun.BlockRunStatus.FAILED,
        ]:
            arr.add(block_uuid)
    return list(arr)
```

由此：
- `completed_block_uuids = _build_block_uuids(self.completed_block_runs)` → 输入仅含 COMPLETED 的 BlockRun → `_build_block_uuids` 过滤后仍只含 COMPLETED → **`completed_block_uuids` 仅含 COMPLETED 的 block_uuid**
- `finished_block_uuids = _build_block_uuids(self.block_runs)` → 输入含全部 BlockRun → 过滤后含 COMPLETED + UPSTREAM_FAILED + FAILED → **`finished_block_uuids` 含所有终态 block_uuid**

**普通 Block（无 dynamic_upstream_block_uuids / dynamic_block_index）**：

普通 Block 的可执行性判断走 `all_upstream_blocks_completed(completed_block_uuids, upstream_block_uuids_override)` 路径（L1209–L1216），只检查 `completed_block_uuids`。由于 `completed_block_uuids` **仅含 COMPLETED**，普通 Block 的上游必须全部 COMPLETED 才可执行。

**当 `allow_blocks_to_fail=True` 时**，失败上游的下游 Block 已被 `update_block_run_statuses()` 标记为 UPSTREAM_FAILED（不再是 INITIAL），因此不会出现在 `executable_block_runs()` 的候选列表中。**普通 Block 在 `allow_blocks_to_fail` 为 True 或 False 时的可执行判断逻辑完全相同**——区别仅在于 False 时 PipelineRun 已终止（不会进入调度），True 时下游被传播为 UPSTREAM_FAILED（跳过依赖判断）。

**动态上游（有 dynamic_upstream_block_uuids + dynamic_block_index 的 BlockRun）**：

这是 `allow_blocks_to_fail` **改变判断集合**的分支（L1115–L1127）：

```python
if allow_blocks_to_fail:
    completed = all(uuid in finished_block_uuids for uuid in uuids_to_check)
else:
    completed = all(uuid in completed_block_uuids for uuid in uuids_to_check)
```

| allow_blocks_to_fail | 检查集合 | 含义 |
|---|---|---|
| False | `completed_block_uuids`（仅 COMPLETED） | 动态上游必须 COMPLETED |
| True | `finished_block_uuids`（COMPLETED + FAILED + UPSTREAM_FAILED） | 动态上游达到终态即可（含 FAILED） |

**动态 Block 子节点（is_dynamic_block_child）**：

调用 `check_all_dynamic_upstreams_completed()`（`mage_ai/data_preparation/models/block/dynamic/utils.py` L1002–L1061），该函数**不考虑 `allow_blocks_to_fail`**，只检查 `BlockRun.BlockRunStatus.COMPLETED`：

```python
selected = [
    br for br in block_runs
    if pipeline.get_block(br.block_uuid).uuid == upstream_block.uuid
    and br.status == BlockRun.BlockRunStatus.COMPLETED
]
if len(list(selected)) < len(combos) + 1:
    return False
```

即使 `allow_blocks_to_fail=True`，动态 Block 子节点的上游若为 FAILED/UPSTREAM_FAILED，该子节点也**不可执行**——必须等上游全部 COMPLETED。

**三层影响总结**：

| 影响层 | allow_blocks_to_fail=False | allow_blocks_to_fail=True |
|---|---|---|
| **入口终止** | 有 Block FAILED → PipelineRun 立即 FAILED | 有 Block FAILED → 不终止，继续调度 |
| **失败传播** | 始终执行（但无实际意义，因已终止） | 始终执行，下游标记 UPSTREAM_FAILED |
| **可执行判断（普通 Block）** | 不到达此步 | 上游必须 COMPLETED；下游已 UPSTREAM_FAILED 不在候选中 |
| **可执行判断（动态上游 BlockRun）** | 检查 `completed_block_uuids`（仅 COMPLETED） | 检查 `finished_block_uuids`（含 FAILED/UPSTREAM_FAILED） |
| **可执行判断（动态 Block 子节点）** | 仅 COMPLETED 算完成 | 仅 COMPLETED 算完成（不受配置影响） |

### 4.6 重试判断

重试发生在 Block 执行器层面，而非调度器层面：

#### 4.6.1 执行器内重试

`mage_ai/data_preparation/executors/block_executor.py` `BlockExecutor.execute()`（L592–L706）：

```python
@retry(
    retries=retry_config.retries if self.RETRYABLE else 0,
    delay=retry_config.delay,
    max_delay=retry_config.max_delay,
    exponential_backoff=retry_config.exponential_backoff,
)
def __execute_with_retry():
    return self._execute(...)
```

- `self.RETRYABLE` 标志决定是否启用重试（大部分 Block 默认为 True）
- 重试期间 `self.retry_metadata` 记录尝试次数，通过 `global_vars['retry']` 传递给 Block 代码
- **重试不改变 BlockRun 状态**——BlockRun 在 `run_block()` 入口已设为 RUNNING，重试期间始终保持 RUNNING
- 重试耗尽后抛出异常，触发 `on_failure` → `on_block_failure()` → BlockRun 标记 FAILED

#### 4.6.2 状态写回重试与崩溃恢复的边界

`on_block_complete()` 和 `on_block_failure()` 中的 DB 写回操作自带 `@retry(retries=2, delay=5)`。这是 **DB 写入重试**，与 Block 业务逻辑重试无关。

**写回重试的职责范围**：
- 保护目标：`BlockRun.update(status=..., metrics=..., completed_at=...)` 这一 DB 写操作
- 重试触发条件：DB 连接超时、事务冲突、临时性 DB 不可用
- 最多 2 次重试，间隔 5 秒，无指数退避
- 重试耗尽后异常抛出，被 `@safe_db_query` 装饰器捕获（静默处理）

#### 4.6.3 崩溃恢复路径（按调度模式分离）

`__fetch_crashed_block_runs()`（L871–L904）检测 RUNNING/QUEUED 状态但对应 Job 已不存在的 BlockRun，将其重置为 INITIAL 以便重跑。不同调度模式的崩溃恢复路径和覆盖范围各不相同：

**路径 A：标准 Batch Pipeline（`__schedule_blocks`）**

触发位置：`__schedule_blocks()`（L624）将 `__fetch_crashed_block_runs()` 的结果拼接到待调度列表头部：

```python
block_runs_to_schedule = self.__fetch_crashed_block_runs() + block_runs_to_schedule
```

恢复条件：`job_manager.has_block_run_job(br.id)` 返回 False
- 检查的是 BLOCK_RUN 类型的 Job（`block_run_{id}`）
- BlockRun 停留在 RUNNING/QUEUED 但对应 Worker 进程已退出 → 重置 INITIAL → 被重新调度执行

**路径 B：单进程 / STREAMING Pipeline（`__schedule_pipeline`）**

触发位置：`__schedule_pipeline()`（L858–L860）在添加 PIPELINE_RUN Job 之前调用：

```python
if PipelineType.STREAMING != self.pipeline.type:
    self.__fetch_crashed_block_runs()
```

恢复条件：同路径 A，检查 BLOCK_RUN 类型的 Job
- STREAMING Pipeline 不调用崩溃恢复（L858 条件排除）
- `run_pipeline_in_one_process=True` 的 Pipeline 走此路径，但崩溃恢复检查的是 BlockRun 级别的 Job，而单进程模式下 Block 在 Pipeline 进程内执行（不创建 BLOCK_RUN Job），因此 `has_block_run_job()` 始终返回 False → 所有 RUNNING/QUEUED 的 BlockRun 都会被重置 INITIAL
- 重置后，`__schedule_pipeline()` 创建新的 PIPELINE_RUN Job 重新执行整 Pipeline

**路径 C：Pipeline Job 崩溃（`has_pipeline_run_job` 守卫）**

`__schedule_pipeline()` 入口检查（L848–L853）：

```python
if job_manager.has_pipeline_run_job(self.pipeline_run.id):
    return    ← Pipeline Job 仍存在，不重新调度
```

- 检查的是 PIPELINE_RUN 类型的 Job（`pipeline_run_{id}`）
- 若 Pipeline 进程仍在运行（`job_dict` 中有记录），则不会创建新 Job，也不会触发崩溃恢复
- 若 Pipeline 进程已退出（`job_dict` 中无记录），则先执行 `__fetch_crashed_block_runs()` 重置残留的 BlockRun，再创建新 Job

**路径对比**：

| 维度 | 标准 Batch（路径 A） | 单进程 Pipeline（路径 B） | Pipeline Job（路径 C） |
|---|---|---|---|
| 恢复触发 | `__schedule_blocks()` 每次调度时 | `__schedule_pipeline()` 入口 | `__schedule_pipeline()` 入口 |
| 检查对象 | `has_block_run_job(br.id)` | `has_block_run_job(br.id)` | `has_pipeline_run_job(pipeline_run.id)` |
| 修复粒度 | 单个 BlockRun → INITIAL | 全部残留 BlockRun → INITIAL | 整 Pipeline 重新执行 |
| 恢复延迟 | 同轮 `schedule()` | 下一轮 `schedule_all()`（最多 10s） | 下一轮 `schedule_all()`（最多 10s） |
| 脏 Job 风险 | Worker 进程残留 → `has_block_run_job` 返回 True → 不重置，需等 300s 心跳 TTL | 同路径 A | Pipeline 进程残留 → `has_pipeline_run_job` 返回 True → 不创建新 Job，需等 300s 心跳 TTL |
| STREAMING | 不适用 | 不执行崩溃恢复（L858 排除） | 同路径 B |

**写回重试与崩溃恢复的边界场景**：

| 场景 | 写回重试 | 崩溃恢复 | BlockRun 最终状态 |
|---|---|---|---|
| DB 临时不可用，1 秒后恢复 | 覆盖（重试成功） | 不需要 | COMPLETED/FAILED（正确） |
| DB 长时间不可用，重试耗尽 | 不覆盖 | 部分覆盖 | RUNNING → INITIAL → 重跑 |
| Worker 进程被 SIGKILL | 不适用 | 覆盖 | RUNNING → INITIAL → 重跑 |
| `@safe_db_query` 吞掉重试耗尽的异常 | 不覆盖 | 覆盖（Job 已不存在时） | RUNNING → INITIAL → 重跑 |
| Job 仍在 `job_dict` 中但实际已僵死 | 不适用 | **不覆盖**（has_job=True） | 停留 RUNNING，需等 300s 心跳 TTL |

#### 4.6.4 Pipeline 级重试

`mage_ai/orchestration/pipeline_scheduler_original.py` `retry_pipeline_run()`（L1398–L1423）创建**全新 PipelineRun**：
- 复用原 `execution_date`、`variables`、`event_variables`、`backfill_id`
- 新建空的 BlockRun 集合（`create_block_runs=False`，在 start() 时创建）
- 不修改原 PipelineRun 的状态，两代运行记录并存

### 4.7 后续调度读取

下一次调度决策依赖以下数据的读取：

#### 4.7.1 读取时机

| 读取点 | 触发条件 | 读取内容 |
|---|---|---|
| `schedule_all()` 主循环 | 每 10 秒 | 所有 ACTIVE PipelineSchedule + RUNNING PipelineRun |
| `PipelineScheduler.schedule()` | Block 完成/超时检测 | 当前 PipelineRun 的全部 BlockRun 状态 |
| `executable_block_runs()` | `schedule()` 内部调用 | BlockRun 状态 + Pipeline DAG 拓扑 |

#### 4.7.2 数据刷新机制

| 方法 | 刷新方式 | 目的 |
|---|---|---|
| `db_connection.session.expire_all()` | `schedule_all()` 入口 | 清空 SQLAlchemy 会话缓存，强制从 DB 重新读取 |
| `pipeline_run.refresh()` | `on_block_complete()`、`schedule()` | 刷新当前 PipelineRun 及其关联的 block_runs |
| `block_run.refresh()` | `schedule()` 入口、`__fetch_crashed_block_runs()` | 逐个刷新 BlockRun 确保读到最新状态 |

#### 4.7.3 闭环链路总结

```
① BlockExecutor._execute() 执行业务逻辑
       ↓
② Block.store_variables() 写入输出变量（磁盘/内存缓存）
       ↓ （失败 → 异常传播 → 走步骤 ④ 失败路径）
③ on_complete/on_failure 回调
       ↓
④ BlockRun.update(status=..., completed_at=..., metrics=...)  写入 DB
       ↓ （@retry 保护，耗尽后依赖崩溃恢复）
⑤ PipelineScheduler.schedule() 重新进入
       ↓
⑥ pipeline_run.refresh() + block_run.refresh()  读取最新 DB 状态
       ↓
⑦ update_block_run_statuses()  传播 FAILED/UPSTREAM_FAILED
       ↓
⑧ executable_block_runs()  基于刷新后的状态判断下游 Block 可执行性
       ↓
⑨ __schedule_blocks()  将可执行 Block 入队
       ↓
⑩ 下游 Block 的 run_block() 从 VariableManager/内存缓存读取上游输出
       ↓ 返回 ①
```

**核心保障与边界**：
- 步骤 ② 失败 → BlockRun 标记 FAILED（非 COMPLETED），不会出现"变量丢失但状态为 COMPLETED"的情况
- 步骤 ④ 的 `@retry` 保证 DB 写入的常见瞬时故障可恢复；耗尽后 `@safe_db_query` 吞异常，BlockRun 停留 RUNNING，由崩溃恢复兜底
- 步骤 ⑥ 的 `refresh()` 保证读一致性，避免读到会话缓存中的过期状态
- 步骤 ⑦ 的失败传播**不受 `allow_blocks_to_fail` 影响**——无论配置如何，上游 FAILED 的下游总是被标记 UPSTREAM_FAILED；`allow_blocks_to_fail` 只控制入口是否终止
- 步骤 ⑧ 的 `completed_block_uuids` **仅含 COMPLETED**，普通 Block 的上游必须全部 COMPLETED 才可执行；动态上游 BlockRun 在 `allow_blocks_to_fail=True` 时改用 `finished_block_uuids`（含 FAILED/UPSTREAM_FAILED）；动态 Block 子节点始终只认 COMPLETED
- 步骤 ⑩ 的变量存储与 DB 状态一致——步骤 ② 成功 + 步骤 ④ 成功时两者一致；步骤 ② 失败时步骤 ④ 走 FAILED 路径，两者仍一致

### 4.8 闭环中的潜在断裂点

| 断裂点 | 风险 | 影响 |
|---|---|---|
| DB 写回 `@retry` 2 次均失败 | BlockRun 状态停留在 RUNNING | 下一轮 `__fetch_crashed_block_runs()` 将其重置为 INITIAL，导致**重跑**；若 Job 仍在 `job_dict` 中则不会被重置，需等 300s 心跳 TTL |
| `schedule_all()` 刷新会话与 `on_block_complete()` 写入时序交错 | 依赖判断读到部分更新状态 | 可能漏调度本应执行的下游 Block（下一轮可自动恢复） |
| `PipelineExecutor` 内存缓存模式下 Block 异常 | `__run_blocks()` 退出，已执行 Block 的输出丢失 | 后续 Block 无法读取上游输出，PipelineRun 停留在 RUNNING，需等下一轮 `schedule()` 修复 |
| CONDITION_FAILED 状态不被 `any_blocks_failed()` 统计 | Pipeline 标记 COMPLETED 但实际有 Block 被跳过 | 用户可能误认为所有 Block 均成功执行 |
| 动态 Block 子节点不支持 `allow_blocks_to_fail` | 上游 FAILED 时动态子节点永远无法获得调度 | Pipeline 无法自动完成，需手动干预或超时终止 |
| 单进程模式下 `has_block_run_job` 始终返回 False | 无 BLOCK_RUN Job 存在，所有 RUNNING/QUEUED BlockRun 均被重置 INITIAL | 崩溃恢复过度重置，可能将本已成功但 DB 写回延迟的 BlockRun 也重置为 INITIAL |

---

## 5. 潜在风险分析

### 5.1 竞态条件

1. **多调度器副本并发**：`schedule_all()` 注释明确指出当前并发限制检查与 API/事件触发之间可能存在竞态，尽管使用分布式锁，但在锁获取前的配额读取仍有窗口期
2. **INITIAL 状态 PipelineRun 无锁保护**：`pipeline_run_{id}` 锁仅在 start() 和 schedule() 中获取，而 INITIAL → RUNNING 的转换依赖 `should_schedule()` 与并发计数之间的非原子操作
3. **Redis 不可用时的退化**：DistributedLock 在 Redis 不可用时直接返回 True（无锁），ProcessQueue.has_job() 也仅依赖本地 job_dict，多实例场景下会重复执行

### 5.2 可靠性风险

1. **崩溃恢复不完整**：`__fetch_crashed_block_runs()` 仅重置 BlockRun 状态，但不清理可能已在队列中的脏 Job；若进程异常退出，Redis 中 job_id 键需等待 300s 心跳 TTL 过期才能清除
2. **状态机缺少 SHUTDOWN 处理**：调度器进程被 SIGTERM 时未对 RUNNING 状态的 PipelineRun/BlockRun 做持久化标记，重启后依赖崩溃恢复逻辑
3. **回调执行异步化**：`on_pipeline_run_cancelled` 通过 GenericJob 异步执行，取消动作与回调之间存在时间差，且 GenericJob 自身失败无重试

### 5.3 性能与扩展性

1. **全局串行调度**：`schedule_all()` 在单线程内按 Pipeline 顺序处理，Pipeline 数量大时单轮耗时增长，无法分片
2. **N+1 查询问题**：`executable_block_runs()` 内对每个 Block 多次调用 `pipeline.get_block()`，Block 数量大时产生大量 DB/IO 查询
3. **Worker Pool 固定大小**：ProcessQueue 的 worker 数在启动时确定，无法按任务类型（CPU/IO 密集）动态伸缩
4. **每轮全量扫描**：调度循环每轮扫描所有 ACTIVE Schedule 和所有 RUNNING PipelineRun，随规模增长线性变慢

### 5.4 可观测性

1. **调度循环自身无指标**：`schedule_all()` 无执行耗时、处理 Pipeline/Block 数等内部指标上报，仅依赖日志
2. **重试元数据不完整**：Block 的实际重试次数、每次重试的错误未结构化存入 metrics，仅通过日志输出
3. **分布式锁获取失败静默**：锁获取失败直接 continue，无告警或统计，调度遗漏难以发现

---

## 6. 后续调查方向

### 6.1 高优先级

1. **分布式一致性验证**：在多副本部署场景下，构造高并发触发（API + 时间调度同时），验证是否会生成重复 PipelineRun 或 Block 被重复执行
2. **Redis 故障演练**：模拟 Redis 不可用/网络分区，观察调度行为是否符合预期（单副本正常、多副本可能重复执行），并评估对业务的影响
3. **崩溃恢复路径测试**：强制 kill 调度进程与 worker 进程，验证 PipelineRun 和 BlockRun 是否能在重启后正确恢复并最终达成终态

### 6.2 中优先级

4. **大规模调度压测**：构造 1000+ PipelineSchedule、单 Pipeline 含 100+ Block 的场景，测量 `schedule_all()` 单轮耗时、数据库连接池压力、内存占用
5. **重试策略实测**：验证 Block 级指数退避在 DB 连接超时、外部 API 抖动等真实故障下的有效性，并统计端到端成功率
6. **并发限制有效性**：验证 `pipeline_run_limit_all_triggers` 在跨 Schedule 场景是否正确生效，检查计数是否包含 INITIAL 状态

### 6.3 低优先级 / 优化方向

7. **调度分片方案**：评估将 PipelineSchedule 按 hash 分片到多个调度实例（或引入调度主节点）以提升水平扩展能力
8. **状态机持久化增强**：为 PipelineRun/BlockRun 增加 `last_heartbeat_at` 字段，由 worker 定期刷新，调度器可更精准地判断僵尸任务
9. **指标埋点**：为调度主循环、锁获取、队列深度、重试次数等关键路径添加 Prometheus 指标
10. **取消流程原子性**：评估将 kill job + 更新 BlockRun 状态 + 触发回调封装为更原子的流程，减少中间状态窗口

---

## 7. 关键代码文件索引

| 模块 | 文件路径 | 核心内容 |
|---|---|---|
| 调度主入口 | `mage_ai/orchestration/pipeline_scheduler_original.py` | schedule_all(), PipelineScheduler 类, run_block/run_pipeline |
| 数据模型 | `mage_ai/orchestration/db/models/schedules.py` | PipelineSchedule, PipelineRun, BlockRun, GenericJob, EventMatcher |
| 调度进程管理 | `mage_ai/server/scheduler_manager.py` | SchedulerManager, LoopTimeTrigger 启动 |
| 触发器实现 | `mage_ai/orchestration/triggers/` | TimeTrigger, LoopTimeTrigger, EventTrigger |
| Job 管理层 | `mage_ai/orchestration/job_manager.py` | JobManager, JobType 枚举 |
| 执行队列实现 | `mage_ai/orchestration/queue/process_queue.py` | ProcessQueue, Worker, poll_job_and_execute |
| 并发配置 | `mage_ai/orchestration/concurrency.py` | ConcurrencyConfig, OnLimitReached |
| 分布式锁 | `mage_ai/orchestration/utils/distributed_lock.py` | DistributedLock（Redis 锁） |
| 重试工具 | `mage_ai/shared/retry.py` | @retry 装饰器（指数退避） |
| 进程执行管理 | `mage_ai/orchestration/execution_process_manager.py` | ExecutionProcessManager（本地进程跟踪） |
| Block 执行器 | `mage_ai/data_preparation/executors/block_executor.py` | BlockExecutor.execute(), _execute(), __update_block_run_status() |
| Pipeline 执行器 | `mage_ai/data_preparation/executors/pipeline_executor.py` | PipelineExecutor.__run_blocks(), 内存缓存输出传递 |
| Block 变量存储 | `mage_ai/data_preparation/models/block/__init__.py` | Block.store_variables(), execute_sync() |
| 服务端配置 | `mage_ai/settings/server.py` | SCHEDULER_TRIGGER_INTERVAL, CONCURRENCY_CONFIG_* |
| 状态检查辅助 | `mage_ai/orchestration/run_status_checker.py` | Sensor 式上游状态查询 |

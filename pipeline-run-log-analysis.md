# Pipeline Run 日志代码分析

## 1. 核心抽象

### 1.1 数据模型层

#### PipelineRun 模型
**文件**：`mage_ai/orchestration/db/models/schedules.py`

`PipelineRun` 是管道运行的核心数据模型，代表一次管道执行实例。

```python
class PipelineRun(PipelineRunProjectPlatformMixin, BaseModel):
    class PipelineRunStatus(StrEnum):
        INITIAL = 'initial'      # 初始状态
        RUNNING = 'running'      # 运行中
        COMPLETED = 'completed'  # 已完成
        FAILED = 'failed'        # 失败
        CANCELLED = 'cancelled'  # 已取消

    # 核心字段
    pipeline_schedule_id = Column(Integer, ForeignKey('pipeline_schedule.id'), index=True)
    pipeline_uuid = Column(String(255), index=True)
    execution_date = Column(DateTime(timezone=True), index=True)
    status = Column(Enum(PipelineRunStatus), default=PipelineRunStatus.INITIAL, index=True)
    started_at = Column(DateTime(timezone=True))
    completed_at = Column(DateTime(timezone=True))
    variables = Column(JSON)
    passed_sla = Column(Boolean, default=False)
    event_variables = Column(JSON)
    metrics = Column(JSON)
    backfill_id = Column(Integer, ForeignKey('backfill.id'), index=True)
    executor_type = Column(Enum(ExecutorType), default=ExecutorType.LOCAL_PYTHON)
```

**核心属性和方法**：
- `execution_partition`：日志分区标识，格式为 `{schedule_id}/{execution_date}`，用于组织日志文件路径
- `logs`：获取管道日志和调度器日志（通过 LoggerManager）
- `block_runs_count` / `completed_block_runs_count`：块运行统计
- `all_blocks_completed(include_failed_blocks)`：检查所有块是否完成
- `any_blocks_failed()`：检查是否有块失败
- `complete()`：标记管道运行为完成状态，**设置 completed_at**
- `create_block_runs()`：根据管道结构创建所有 BlockRun 实例
- `executable_block_runs(allow_blocks_to_fail)`：计算当前可执行的块列表
- `update_block_run_statuses(block_runs)`：批量更新块运行状态（处理上游失败、条件失败的传播）

#### BlockRun 模型
**文件**：`mage_ai/orchestration/db/models/schedules.py`

`BlockRun` 表示管道中单个块的执行实例。

```python
class BlockRun(BlockRunProjectPlatformMixin, BaseModel):
    class BlockRunStatus(StrEnum):
        INITIAL = 'initial'           # 初始状态
        QUEUED = 'queued'             # 已入队等待执行
        RUNNING = 'running'           # 正在执行
        COMPLETED = 'completed'       # 成功完成
        FAILED = 'failed'             # 执行失败
        CANCELLED = 'cancelled'       # 被取消
        UPSTREAM_FAILED = 'upstream_failed'    # 上游块失败导致未执行
        CONDITION_FAILED = 'condition_failed'  # 条件块不满足导致未执行

    pipeline_run_id = Column(Integer, ForeignKey('pipeline_run.id'), index=True)
    block_uuid = Column(String(255))
    status = Column(Enum(BlockRunStatus), default=BlockRunStatus.INITIAL)
    started_at = Column(DateTime(timezone=True))
    completed_at = Column(DateTime(timezone=True))
    metrics = Column(JSON)
```

**核心属性和方法**：
- `logs`：获取块运行日志（通过 LoggerManager）
- `batch_update_status(block_run_ids, status)`：批量更新块运行状态
- `pipeline_run`：关联的 PipelineRun 实例

### 1.2 日志管理层

#### LoggerManager
**文件**：`mage_ai/data_preparation/logging/logger_manager.py`

日志管理核心类，负责日志的创建、存储和检索。

**核心功能**：
- 支持文件日志（RotatingFileHandler）和流日志（StreamHandler）两种模式
- 文件日志最大 20MB，保留 10 个备份
- 按 `pipeline_uuid/partition/block_uuid` 分层组织日志文件
- 支持日志保留期配置，自动清理旧日志
- 异步日志读取支持（get_logs_async）

**日志路径结构**：
```
logs/
  {pipeline_uuid}/
    {schedule_id}/
      {execution_date}/
        {block_uuid}.log      # 块运行日志
        scheduler.log          # 调度器日志
        pipeline.log           # 管道运行日志
```

#### LoggerManagerFactory
**文件**：`mage_ai/data_preparation/logging/logger_manager_factory.py`

工厂类，根据配置创建不同类型的日志管理器。

```python
class LoggerManagerFactory:
    @classmethod
    def get_logger_manager(self, repo_config: RepoConfig = None, **kwargs):
        if repo_config is not None and repo_config.logging_config is not None:
            logger_type = repo_config.logging_config.get('type')
            if logger_type == LoggerType.S3:
                return S3LoggerManager(...)
            elif logger_type == LoggerType.GCS:
                return GCSLoggerManager(...)
        return LoggerManager(...)
```

支持的日志存储后端：
- 本地文件（默认）
- S3（S3LoggerManager）
- GCS（GCSLoggerManager）

#### DictLogger
**文件**：`mage_ai/data_preparation/logging/logger.py`

结构化日志包装器，将日志消息编码为 JSON 格式，便于日志分析和检索。

```python
class DictLogger():
    def __init__(self, logger: logging.Logger, logging_tags: Dict = None):
        self.logger = logger
        self.logging_tags = logging_tags or dict()

    def __send_message(self, method_name, message, error=None, log_level=None, **kwargs):
        now = datetime.utcnow()
        data = dict(
            level=...,
            message=message,
            timestamp=now.timestamp(),
            uuid=uuid.uuid4().hex,
        )
        if error:
            data['error'] = traceback.format_exc()
            data['error_stack'] = traceback.format_stack()
            data['error_stacktrace'] = str(error)

        msg = simplejson.dumps(merge_dict(self.logging_tags, merge_dict(kwargs, data)), ...)
        getattr(self.logger, method_name)(msg)
```

**日志格式**（JSON 行）：
```json
{
  "level": "INFO",
  "message": "Execute PipelineRun 123",
  "timestamp": 1718562466.123,
  "uuid": "abc123def456...",
  "pipeline_run_id": 123,
  "pipeline_uuid": "my_pipeline",
  "block_run_id": 456,
  "block_uuid": "load_data",
  "hostname": "worker-01"
}
```

### 1.3 调度层

#### PipelineScheduler
**文件**：`mage_ai/orchestration/pipeline_scheduler_original.py`

管道运行的调度核心，负责协调管道和块的执行，管理状态转换。

```python
class PipelineScheduler:
    def __init__(self, pipeline_run: PipelineRun) -> None:
        self.pipeline_run = pipeline_run
        self.pipeline_schedule = pipeline_run.pipeline_schedule
        self.pipeline = get_pipeline_from_platform(...)
        self.streams = [...]  # INTEGRATION 类型管道的流列表

        # 日志
        self.logger_manager = LoggerManagerFactory.get_logger_manager(
            pipeline_uuid=self.pipeline.uuid,
            filename='scheduler.log',
            partition=self.pipeline_run.execution_partition,
            repo_config=self.pipeline.repo_config,
        )
        self.logger = DictLogger(self.logger_manager.logger)

        # 通知
        self.notification_sender = NotificationSender(...)

        # 并发
        self.concurrency_config = ConcurrencyConfig.load(...)
        self.allow_blocks_to_fail = ...
```

**核心方法**：
- `start()`：启动管道运行
- `stop()`：停止管道运行
- `schedule()`：调度块执行（主调度循环）
- `on_block_complete(block_uuid, metrics)`：块完成回调
- `on_block_failure(block_uuid, **kwargs)`：块失败回调
- `on_pipeline_run_failure(error_msg, status)`：管道失败处理
- `__schedule_blocks()`：调度块运行（批量管道）
- `__schedule_integration_streams()`：调度集成流
- `__schedule_pipeline()`：调度整个管道（单进程模式）
- `__check_pipeline_run_timeout()`：检查管道超时
- `__check_block_run_timeout()`：检查块超时
- `__fetch_crashed_block_runs()`：处理崩溃的块运行
- `__run_heartbeat()`：心跳检测

---

## 2. 状态切换

### 2.1 PipelineRun 状态机

```
INITIAL
   │
   ▼
RUNNING ────────────────────────┐
   │                            │
   ├──────────────────────────► COMPLETED  (所有块成功完成)
   │                            │
   ├──────────────────────────► FAILED     (块失败/管道超时)
   │                            │
   └──────────────────────────► CANCELLED  (用户取消/内存超限/超时配置为取消)
```

**状态切换触发点**：

| 源状态 | 目标状态 | 触发位置 | 触发条件 | 关键操作 |
|--------|---------|---------|---------|---------|
| INITIAL | RUNNING | `PipelineScheduler.start()` L197-200 | 成功初始化块运行，获取分布式锁 | 设置 `started_at`，更新状态 |
| RUNNING | COMPLETED | `PipelineScheduler.schedule()` L262 | `all_blocks_completed() == True` 且 `any_blocks_failed() == False` | 调用 `complete()`，**设置 `completed_at`**，发送成功通知 |
| RUNNING | FAILED | `PipelineScheduler.schedule()` L251-253 | 所有块完成但存在失败块 | **设置 `completed_at`**，更新状态，调用 `on_pipeline_run_failure()` |
| RUNNING | FAILED | `PipelineScheduler.schedule()` L309 | 块失败且 `allow_blocks_to_fail == False` | **不设置 `completed_at`**，更新状态，调用 `on_pipeline_run_failure()` |
| RUNNING | FAILED/CANCELLED | `PipelineScheduler.schedule()` L306 | 管道运行超时（状态由 `timeout_status` 决定，默认 FAILED） | **不设置 `completed_at`**，更新状态，调用 `on_pipeline_run_failure()` |
| RUNNING | FAILED | `PipelineScheduler.start()` L187 | 初始化块运行时异常 | **不设置 `completed_at`**，更新状态，直接发送失败通知 |
| RUNNING | CANCELLED | `stop_pipeline_run()` L1452 | 用户主动取消 / 内存超限 | **不设置 `completed_at`**，更新状态，取消所有块，终止作业 |
| RUNNING | FAILED/CANCELLED | `StreamingPipelineExecutor.__update_pipeline_run_status()` L288-289 | Streaming 管道结束 | **始终设置 `completed_at`** |

### 2.2 BlockRun 状态机

```
                    ┌───────────────────────────┐
                    │                           │
                    ▼                           │
INITIAL ────────► QUEUED ─────────────────► RUNNING ────────► COMPLETED
  │                │                           │
  │                │                           ├────────────► FAILED
  │                │                           │
  │                │                           └────────────► CANCELLED
  │                │
  ├────────────────┴───────────────────────────────────────► UPSTREAM_FAILED
  │                                                          (上游失败导致跳过)
  │
  └───────────────────────────────────────────────────────► CONDITION_FAILED
                                                          (条件块不满足导致跳过)
```

**状态切换触发点**：

1. **INITIAL → QUEUED**
   - 位置：`PipelineScheduler.__schedule_blocks()` L641-643
   - 条件：块是可执行的（所有上游完成），且有并发配额
   - 操作：更新状态为 QUEUED，添加作业到 JobManager

2. **QUEUED → RUNNING**
   - 位置：`run_block()` L1279-L1283
   - 条件：作业被 JobManager 消费，块状态为 INITIAL/QUEUED/RUNNING
   - 操作：设置 `started_at`，更新状态为 RUNNING

3. **RUNNING → COMPLETED**
   - 位置：`PipelineScheduler.on_block_complete()` L390-394
   - 操作：**设置 `completed_at`**，更新 metrics，触发下一轮 schedule()

4. **RUNNING → FAILED**
   - 位置：`PipelineScheduler.on_block_failure()` L449-453
   - 触发：块执行异常 / 块超时
   - 操作：**不设置 `completed_at`**，记录错误到 `metrics.error`，更新状态

5. **INITIAL → UPSTREAM_FAILED**
   - 位置：`PipelineRun.update_block_run_statuses()` L1223-L1301
   - 条件：上游块失败（FAILED 或 UPSTREAM_FAILED）
   - 说明：这是一个递归传播过程

6. **INITIAL → CONDITION_FAILED**
   - 位置：`PipelineRun.update_block_run_statuses()` L1223-L1301
   - 条件：上游条件块状态为 CONDITION_FAILED

7. **批量 INITIAL/QUEUED/RUNNING → CANCELLED**
   - 位置：`cancel_block_runs_and_jobs()` L1490-L1493
   - 操作：**不设置 `completed_at`**，批量更新状态，终止相关作业

8. **RUNNING/QUEUED → INITIAL（崩溃恢复）**
   - 位置：`PipelineScheduler.__fetch_crashed_block_runs()` L871-L904
   - 条件：块状态为 RUNNING 或 QUEUED，但对应的作业已不存在
   - 操作：重置状态为 INITIAL，以便重新调度

### 2.3 状态检查机制
**文件**：`mage_ai/orchestration/run_status_checker.py`

提供传感器（Sensor）场景下的状态检查功能。

```python
PIPELINE_FAILURE_STATUSES = [
    PipelineRun.PipelineRunStatus.CANCELLED,
    PipelineRun.PipelineRunStatus.FAILED,
]

BLOCK_FAILURE_STATUSES = [
    BlockRun.BlockRunStatus.CANCELLED,
    BlockRun.BlockRunStatus.FAILED,
]
```

`check_status()` 函数根据 pipeline_uuid 和 execution_date 查找最近的运行，检查其状态。

---

## 3. 调度流程详解

### 3.1 块入队到执行的完整顺序

**主调度循环：`PipelineScheduler.schedule()`**

```
schedule()
  │
  ├─ 1. 获取分布式锁（pipeline_run_xxx），超时 10 秒
  ├─ 2. 发送心跳（__run_heartbeat）
  │      └─ 检查内存使用率，超过 95% 则停止管道
  │
  ├─ 3. 刷新所有 BlockRun 状态
  │
  ├─ 4. 检查所有块是否完成（all_blocks_completed）
  │   ├─ 是 → 根据是否有失败标记为 COMPLETED / FAILED，退出
  │   └─ 否 → 继续
  │
  ├─ 5. 检查管道超时（__check_pipeline_run_timeout）
  │   └─ 超时 → 标记 FAILED/CANCELLED，退出
  │
  ├─ 6. 检查是否有块失败且不允许失败
  │   └─ 是 → 标记 FAILED，退出
  │
  └─ 7. 调度块执行（根据管道类型）
      ├─ STREAMING 类型 → __schedule_pipeline()
      ├─ INTEGRATION 类型 → __schedule_integration_streams()
      ├─ 单进程模式 → __schedule_pipeline()
      └─ 普通批处理 → 先检查块超时，再 __schedule_blocks()
```

**块调度详细流程：`__schedule_blocks()`**

```
__schedule_blocks()
  │
  ├─ 1. 更新初始块的状态（传播上游失败 / 条件失败）
  │      update_block_run_statuses(initial_block_runs)
  │
  ├─ 2. 计算可执行的块列表
  │      executable_block_runs = pipeline_run.executable_block_runs(...)
  │
  ├─ 3. 处理崩溃的块运行（作业丢失但状态还在 RUNNING/QUEUED）
  │      crashed_runs = __fetch_crashed_block_runs()
  │      block_runs_to_schedule = crashed_runs + executable_block_runs
  │
  ├─ 4. 应用并发限制
  │      block_run_quota = block_run_limit - len(queued_or_running_block_runs)
  │      if block_run_quota <= 0: return
  │
  └─ 5. 遍历可调度块，逐个入队（前 block_run_quota 个）
           for b in block_runs_to_schedule[:block_run_quota]:
               ├─ 更新状态为 QUEUED
               └─ job_manager.add_job(JobType.BLOCK_RUN, b.id, run_block, ...)
```

**块执行流程：`run_block()`**

```
run_block(pipeline_run_id, block_run_id, variables, tags, ...)
  │
  ├─ 1. 检查 PipelineRun 状态
  │      必须为 RUNNING，否则直接返回
  │
  ├─ 2. 检查 BlockRun 状态
  │      必须为 INITIAL / QUEUED / RUNNING，否则直接返回
  │
  ├─ 3. 更新 BlockRun 状态为 RUNNING
  │      首次运行时设置 started_at
  │
  ├─ 4. 创建 PipelineScheduler 和 BlockExecutor
  │
  └─ 5. 执行块
       ├─ 成功 → on_block_complete()
       │      ├─ 更新状态为 COMPLETED
       │      ├─ 设置 completed_at
       │      ├─ 合并 metrics
       │      └─ 触发下一轮 schedule()（如果 schedule_after_complete）
       └─ 失败 → on_block_failure()
              ├─ 记录错误信息到 metrics.error
              ├─ 更新状态为 FAILED
              └─ 不允许失败时，下次 schedule() 会终止管道
```

### 3.2 QUEUED 状态的作用

QUEUED 状态是块从"可执行"到"实际运行"之间的中间状态，主要作用：

1. **并发控制**：通过统计 QUEUED + RUNNING 的块数量，实现并发数限制
2. **作业追踪**：标记该块已经被提交到作业队列，避免重复提交
3. **崩溃恢复**：服务重启后，通过检查 QUEUED/RUNNING 状态的块是否有对应作业，实现崩溃检测

### 3.3 块入队的唯一来源

块运行进入 QUEUED 状态只有一个来源：**`PipelineScheduler.__schedule_blocks()` 方法中的批量入队逻辑**（L641-643）。

此外还有对应的 project platform 版本（`pipeline_scheduler_project_platform.py` L609-611），逻辑相同。

---

## 4. 失败、取消、超时的路径对比

### 4.1 四种终止路径总览对比

| 维度 | 运行中块失败 | 全部块结束后发现失败 | 管道超时 | 用户取消 |
|------|-------------|-------------------|---------|---------|
| **触发条件** | `allow_blocks_to_fail=False` 且存在失败块 | `all_blocks_completed()=True` 且 `any_blocks_failed()=True` | 运行时间 > `PipelineSchedule.timeout` | 调用 `stop_pipeline_run()` / 内存超限 |
| **触发位置** | `schedule()` L308-L331 | `schedule()` L250-L260 | `schedule()` L302-L307 | `stop_pipeline_run()` L1426-L1457 |
| **最终状态** | FAILED | FAILED | FAILED（默认）或 CANCELLED | CANCELLED（默认） |
| **completed_at** | ❌ **未设置** | ✅ **已设置** | ❌ **未设置** | ❌ **未设置** |
| **失败通知** | ✅ 发送（通过 `on_pipeline_run_failure`） | ✅ 发送（通过 `on_pipeline_run_failure`） | ✅ 仅当最终状态为 FAILED 时发送 | ❌ **不发送** |
| **块取消** | ✅ 调用 `cancel_block_runs_and_jobs()` | ✅ 调用（实际无需要取消的块） | ✅ 调用 `cancel_block_runs_and_jobs()` | ✅ 调用 `cancel_block_runs_and_jobs()` |
| **调用 on_pipeline_run_failure** | ✅ 是，默认 status=FAILED | ✅ 是，默认 status=FAILED | ✅ 是，status 参数为最终状态 | ❌ **否，直接取消** |
| **块状态分布** | 失败块为 FAILED，其余 INITIAL/QUEUED/RUNNING → CANCELLED | 失败块为 FAILED，其余为 COMPLETED/UPSTREAM_FAILED/CONDITION_FAILED | 运行中的块 → CANCELLED | 所有 INITIAL/QUEUED/RUNNING → CANCELLED |

### 4.2 路径一：运行中块失败（allow_blocks_to_fail=False）

**触发时机**：在 `schedule()` 循环中检测到有块已失败，且 `allow_blocks_to_fail == False`。

**代码执行顺序**（`schedule()` L308-L331）：

```python
# 1. 检测条件
elif self.pipeline_run.any_blocks_failed() and not self.allow_blocks_to_fail:

    # 2. 更新管道状态 —— 注意：没有设置 completed_at
    self.pipeline_run.update(status=PipelineRun.PipelineRunStatus.FAILED)

    # 3. 构造错误信息
    failed_block_runs = self.pipeline_run.failed_block_runs
    error_msg = 'Failed blocks: ' + ', '.join([b.block_uuid for b in failed_block_runs]) + '.'

    # 4. 调用 on_pipeline_run_failure，默认 status=FAILED
    self.on_pipeline_run_failure(error_msg)
```

**`on_pipeline_run_failure()` 的执行**（L341-L374）：

```python
def on_pipeline_run_failure(self, error_msg, status=PipelineRun.PipelineRunStatus.FAILED):
    # 1. 记录使用统计
    UsageStatisticLogger().pipeline_run_ended_sync(self.pipeline_run)

    # 2. 仅当 status == FAILED 时发送失败通知
    if status == PipelineRun.PipelineRunStatus.FAILED:
        # 收集失败块的错误堆栈（最多 50 行，超出截断）
        ...
        self.notification_sender.send_pipeline_run_failure_message(...)

    # 3. 取消所有进行中的块运行 + 终止作业
    cancel_block_runs_and_jobs(self.pipeline_run, self.pipeline)
```

**`cancel_block_runs_and_jobs()` 的执行**（L1460-L1519）：

```python
def cancel_block_runs_and_jobs(pipeline_run, pipeline):
    # 1. 收集需要取消的块：状态为 INITIAL/QUEUED/RUNNING
    block_runs_to_cancel = [b for b in pipeline_run.block_runs
                            if b.status in [INITIAL, QUEUED, RUNNING]]

    # 2. 批量更新为 CANCELLED
    BlockRun.batch_update_status(cancelled_block_run_ids, BlockRun.BlockRunStatus.CANCELLED)

    # 3. 终止相关作业
    #    - INTEGRATION/STREAMING：kill_pipeline_run_job + kill_integration_stream_job
    #    - 普通批处理：kill_block_run_job（每个运行中的块）
    #    - K8S 执行器：executor.cancel()

    # 4. 入队取消清理作业
    GenericJob.enqueue_cancel_pipeline_run(pipeline_run.id, cancelled_block_run_ids)
```

### 4.3 路径二：全部块结束后发现失败

**触发时机**：所有块都已到达终态（COMPLETED / FAILED / UPSTREAM_FAILED / CONDITION_FAILED），但其中存在 FAILED 状态的块。通常出现在 `allow_blocks_to_fail=True` 的场景下。

**代码执行顺序**（`schedule()` L240-L260）：

```python
# 1. 检测条件：所有块完成（allow_blocks_to_fail=True 时 FAILED 也算完成）
if self.pipeline_run.all_blocks_completed(self.allow_blocks_to_fail):

    # 2. 检测是否存在失败块
    if self.pipeline_run.any_blocks_failed():

        # 3. 更新管道状态 —— 注意：这里设置了 completed_at
        self.pipeline_run.update(
            status=PipelineRun.PipelineRunStatus.FAILED,
            completed_at=datetime.now(tz=pytz.UTC),  # ✅ 已设置
        )

        # 4. 构造错误信息
        failed_block_runs = self.pipeline_run.failed_block_runs
        error_msg = 'Failed blocks: ' + ', '.join([b.block_uuid for b in failed_block_runs]) + '.'

        # 5. 调用 on_pipeline_run_failure（发送通知 + 取消块，但实际没有需要取消的块）
        self.on_pipeline_run_failure(error_msg)
```

**与路径一的关键区别**：
- **completed_at 已设置**：因为所有块都已结束，可以确定结束时间
- **实际块取消无效**：`cancel_block_runs_and_jobs()` 会被调用，但此时所有块都已经是终态，没有 INITIAL/QUEUED/RUNNING 状态的块需要取消

### 4.4 路径三：管道超时

**触发时机**：`PipelineSchedule.timeout` 配置了超时秒数，且管道运行时间超过该阈值。

**代码执行顺序**（`schedule()` L302-L307）：

```python
# 1. 检测超时
elif self.__check_pipeline_run_timeout():

    # 2. 确定最终状态：由 timeout_status 配置决定，默认 FAILED
    status = (
        self.pipeline_schedule.timeout_status or PipelineRun.PipelineRunStatus.FAILED
    )

    # 3. 更新管道状态 —— 注意：没有设置 completed_at
    self.pipeline_run.update(status=status)

    # 4. 调用 on_pipeline_run_failure，传入最终状态
    self.on_pipeline_run_failure('Pipeline run timed out.', status=status)
```

**`on_pipeline_run_failure()` 的差异**：
- 如果 `status == FAILED`：发送失败通知 + 取消块
- 如果 `status == CANCELLED`：**不发送通知**，只取消块

**块超时 vs 管道超时的区别**：

| 维度 | 块超时 | 管道超时 |
|------|-------|---------|
| 触发位置 | `__check_block_run_timeout()` | `__check_pipeline_run_timeout()` |
| 配置位置 | 块的 `timeout` 属性 | `PipelineSchedule.timeout` |
| 最终块状态 | FAILED | 未完成的块 → CANCELLED |
| 最终管道状态 | 取决于 allow_blocks_to_fail | FAILED / CANCELLED |
| 通知 | 管道失败时发送 | 取决于最终状态 |

### 4.5 路径四：用户取消 / 内存超限

**触发时机**：
1. 用户通过 API 主动调用取消
2. 心跳检测发现内存使用率 ≥ 95%（仅 LOCAL_PYTHON 执行器）

**代码执行顺序**（`stop_pipeline_run()` L1426-L1457）：

```python
def stop_pipeline_run(pipeline_run, pipeline=None, status=PipelineRun.PipelineRunStatus.CANCELLED):
    # 1. 前置检查：仅允许取消 INITIAL 或 RUNNING 状态
    if pipeline_run.status not in [INITIAL, RUNNING]:
        return

    # 2. 更新管道状态 —— 注意：没有设置 completed_at
    pipeline_run.update(status=status)  # 默认 CANCELLED

    # 3. 记录使用统计
    UsageStatisticLogger().pipeline_run_ended_sync(pipeline_run)

    # 4. 取消所有块运行 + 终止作业（不通过 on_pipeline_run_failure）
    cancel_block_runs_and_jobs(pipeline_run, pipeline)
```

**与其他路径的关键区别**：
- **不调用 `on_pipeline_run_failure()`**：直接跳过，不经过该函数
- **不发送失败通知**：因为不调用 `on_pipeline_run_failure()`，且状态为 CANCELLED
- **不设置 completed_at**：与路径一、三一致

### 4.6 特殊情况：start() 初始化失败

**触发时机**：`PipelineScheduler.start()` 中创建 BlockRun 时抛出异常。

**代码执行顺序**（`start()` L176-L193）：

```python
try:
    # 初始化块运行...
except Exception as e:
    error_msg = 'Fail to initialize block runs.'
    self.logger.exception(error_msg, ...)

    # 更新管道状态 —— 注意：没有设置 completed_at
    self.pipeline_run.update(status=PipelineRun.PipelineRunStatus.FAILED)

    # 直接发送失败通知（不通过 on_pipeline_run_failure）
    self.notification_sender.send_pipeline_run_failure_message(
        pipeline=self.pipeline,
        pipeline_run=self.pipeline_run,
        error=error_msg,
    )
    return False
```

**特点**：
- 不调用 `on_pipeline_run_failure()`，直接发送通知
- 不调用 `cancel_block_runs_and_jobs()`（块尚未创建成功）
- 不设置 completed_at

### 4.7 特殊情况：Streaming 管道

**文件**：`mage_ai/data_preparation/executors/streaming_pipeline_executor.py`

Streaming 管道的状态更新走独立路径：

```python
def __update_pipeline_run_status(self, pipeline_run_id, status, error=None):
    pipeline_run = PipelineRun.query.get(pipeline_run_id)

    # 注意：始终设置 completed_at
    pipeline_run.update(
        status=status,
        completed_at=datetime.now(tz=pytz.UTC),  # ✅ 始终设置
    )

    if status == PipelineRun.PipelineRunStatus.FAILED:
        # 记录统计 + 发送通知
        notification_sender.send_pipeline_run_failure_message(...)
```

**特点**：无论 FAILED 还是其他终止状态，**始终设置 `completed_at`**。

---

## 5. 边界处理

### 5.1 并发控制

#### 分布式锁
`PipelineScheduler.start()` 和 `schedule()` 使用分布式锁防止并发调度冲突：

```python
lock_key = f'pipeline_run_{self.pipeline_run.id}'
if not lock.try_acquire_lock(lock_key):  # schedule() 中超时 10 秒
    return
try:
    # 调度逻辑...
finally:
    lock.release_lock(lock_key)
```

#### 块并发限制
`ConcurrencyConfig.block_run_limit` 控制最大并发块数：
- 统计当前 QUEUED + RUNNING 的块数量
- 配额 = 限制数 - 当前进行中数量
- 配额 ≤ 0 时不调度新块

### 5.2 崩溃恢复

`__fetch_crashed_block_runs()` 处理服务崩溃后的恢复：
1. 遍历所有状态为 RUNNING 或 QUEUED 的块
2. 检查对应的作业是否还存在
3. 作业不存在的块重置为 INITIAL 状态
4. 这些块会被重新调度

### 5.3 内存保护

`__run_heartbeat()` 中检查系统内存使用率：
- 阈值：95%（MEMORY_USAGE_MAXIMUM）
- 仅对 LOCAL_PYTHON 执行器生效
- 超限调用 `memory_usage_failure()` → `stop()` → `stop_pipeline_run()`
- 发送内存超限通知

### 5.4 日志边界

#### 日志轮转
- 最大文件大小：20MB（MAX_LOG_FILE_SIZE）
- 备份文件数：10个（backupCount）
- 使用 Python 标准库 RotatingFileHandler

#### 日志清理
`LoggerManager.delete_old_logs()`：
- 根据 `retention_period` 配置清理过期日志
- 按 partition（执行日期）进行清理判断
- Streaming 管道日志跳过清理（持续运行）
- 清理异常会被捕获，不影响主流程

### 5.5 重试机制

块状态更新使用 `@retry` 装饰器保证可靠性：

```python
@retry(retries=2, delay=5)
def update_status(metrics=metrics):
    block_run.update(
        status=BlockRun.BlockRunStatus.COMPLETED,
        completed_at=datetime.now(tz=pytz.UTC),
        metrics=metrics_prev,
    )
```

用于 `on_block_complete()` 和 `on_block_failure()` 中的状态更新。

### 5.6 安全数据库查询

使用 `@safe_db_query` 装饰器保护数据库操作，自动处理数据库连接异常和会话管理。

---

## 6. 关键交互流程

### 6.1 管道启动完整流程

```
1. 创建 PipelineRun（INITIAL 状态）
   │
   ▼
2. PipelineScheduler.start()
   ├─ 检查状态（已是 RUNNING 则直接返回）
   ├─ 获取分布式锁
   ├─ 检查是否已有 BlockRun
   │   └─ 没有则创建：
   │       - INTEGRATION 类型：initialize_state_and_runs()
   │       - 其他类型：pipeline_run.create_block_runs()
   ├─ 异常 → FAILED + 直接发送通知 + 返回 False
   ├─ 释放锁
   ├─ 更新 PipelineRun：started_at + RUNNING
   └─ 调用 schedule() 开始调度
       │
       ▼
3. schedule() 循环（每次块完成或定时触发）
   ├─ 心跳检测
   ├─ 检查完成 / 超时 / 失败
   └─ 调度新块 → __schedule_blocks()
       ├─ 更新上游失败状态
       ├─ 计算可执行块
       ├─ 崩溃恢复
       ├─ 并发限制
       └─ 块入队 → QUEUED + job_manager.add_job()
           │
           ▼
4. JobManager 消费作业 → run_block()
   ├─ 状态检查
   ├─ 更新为 RUNNING
   └─ BlockExecutor.execute()
       ├─ 成功 → on_block_complete() → schedule()
       └─ 失败 → on_block_failure() → 下次 schedule() 可能终止管道
```

### 6.2 日志收集流程

```
DictLogger.info(message, **tags)
    │
    ▼
__send_message(method_name, message, error=None, **kwargs)
    ├─ 构造基础数据（level, message, timestamp, uuid）
    ├─ error 时添加堆栈信息
    ├─ 合并 logging_tags + kwargs + data
    ├─ simplejson.dumps 序列化
    └─ 调用底层 logger.{level}(msg)
        │
        ▼
LoggerManager 的 Handler
    ├─ RotatingFileHandler → 写入日志文件
    └─ StreamHandler → 写入内存流（有 destination_config 时）
```

### 6.3 取消流程

```
用户取消 / 内存超限 / ...
    │
    ▼
stop_pipeline_run(pipeline_run, pipeline, status=CANCELLED)
    ├─ 检查状态（非 INITIAL/RUNNING 则直接返回）
    ├─ 更新管道状态为 CANCELLED（不设置 completed_at）
    ├─ 记录使用统计
    └─ cancel_block_runs_and_jobs(pipeline_run, pipeline)
        ├─ 收集 INITIAL/QUEUED/RUNNING 状态的块
        ├─ 批量更新为 CANCELLED
        ├─ 终止作业
        │   ├─ INTEGRATION/STREAMING：kill_pipeline_run_job + kill_integration_stream_job
        │   ├─ 普通批处理：kill_block_run_job（每个运行中块）
        │   └─ K8S 执行器：executor.cancel()
        └─ enqueue_cancel_pipeline_run（后续清理）
```

---

## 7. 核心类关系图

```
PipelineRun (1)
  │
  ├─ status: PipelineRunStatus (INITIAL/RUNNING/COMPLETED/FAILED/CANCELLED)
  ├─ started_at, completed_at  ← 注意：非所有终止路径都设置 completed_at
  ├─ variables, metrics (JSON)
  ├─ execution_partition → 日志路径标识
  │
  ├─ 包含 N 个 BlockRun
  │   ├─ status: BlockRunStatus (8种状态)
  │   ├─ started_at, completed_at
  │   └─ metrics: JSON (error, records, etc.)
  │
  └─ logs 属性 → 通过 LoggerManager 获取日志


PipelineScheduler
  ├─ pipeline_run: PipelineRun
  ├─ pipeline: Pipeline
  ├─ logger: DictLogger
  ├─ logger_manager: LoggerManager
  ├─ notification_sender: NotificationSender
  └─ concurrency_config: ConcurrencyConfig


LoggerManagerFactory
  └─ get_logger_manager() → 创建 LoggerManager 或子类
      ├─ LoggerManager (本地文件)
      ├─ S3LoggerManager (S3 存储)
      └─ GCSLoggerManager (GCS 存储)

DictLogger
  └─ 包装 logging.Logger，输出结构化 JSON 日志
```

---

## 8. 容易混淆的结论澄清

### ❌ 之前容易混淆的结论

| 混淆点 | 错误理解 | 正确结论 |
|--------|---------|---------|
| completed_at | 所有终止状态都会设置 completed_at | **只有全部块结束后失败、正常完成、Streaming 管道**才设置；运行中失败、超时、用户取消**均不设置** |
| 失败通知 | 只要终止就发送通知 | **只有最终状态为 FAILED**时才发送；CANCELLED 不发送；用户取消不经过 on_pipeline_run_failure |
| on_pipeline_run_failure | 所有失败路径都调用 | 用户取消（stop_pipeline_run）和 start() 初始化失败**不调用**，直接处理 |
| 管道超时 | 超时一定是 FAILED 状态 | 由 `timeout_status` 配置决定，**默认 FAILED，可配置为 CANCELLED** |
| 块取消 | 只有用户取消才取消块 | 所有失败路径（运行中块失败、超时）都会调用 `cancel_block_runs_and_jobs()` 取消进行中的块 |
| completed_at 缺失 | 不设置 completed_at 是 bug | 目前代码设计如此，但**存在不一致性**—— Streaming 管道始终设置，非 Streaming 管道部分路径不设置 |

### ✅ 终止路径 completed_at 设置情况速查表

| 终止路径 | completed_at | 代码位置 |
|---------|-------------|---------|
| 正常完成 (COMPLETED) | ✅ 设置 | `PipelineRun.complete()` L1408 |
| 全部块结束后失败 (FAILED) | ✅ 设置 | `schedule()` L253 |
| 运行中块失败 (FAILED) | ❌ 未设置 | `schedule()` L309 |
| 管道超时 (FAILED/CANCELLED) | ❌ 未设置 | `schedule()` L306 |
| 用户取消 (CANCELLED) | ❌ 未设置 | `stop_pipeline_run()` L1452 |
| start() 初始化失败 (FAILED) | ❌ 未设置 | `start()` L187 |
| Streaming 管道任意终止 | ✅ 始终设置 | `streaming_pipeline_executor.py` L289 |

---

## 9. 设计亮点与可改进点

### 9.1 设计亮点

1. **分层日志架构**：工厂模式支持多种日志存储后端（本地、S3、GCS），扩展性好
2. **结构化日志**：JSON 格式便于日志分析、检索和监控集成
3. **完整的状态机**：覆盖了正常、失败、取消、上游失败、条件失败等多种场景
4. **崩溃恢复机制**：通过作业状态与块状态比对实现自动恢复
5. **分布式锁**：防止并发调度冲突，保证状态一致性
6. **细粒度并发控制**：支持管道级别的块并发数限制
7. **丰富的可观测性**：metrics 字段支持收集详细的运行指标

### 9.2 可改进点

**问题1：completed_at 设置不一致**

非 Streaming 管道的多条终止路径不设置 `completed_at`，导致：
- 无法准确计算运行时长
- 数据统计不完整
- Streaming 与非 Streaming 管道行为不一致

涉及位置：
- `schedule()` L309（运行中块失败）
- `schedule()` L306（管道超时）
- `stop_pipeline_run()` L1452（用户取消）
- `start()` L187（初始化失败）

建议统一在所有终止路径设置 `completed_at`。

**问题2：取消时没有超时通知的选项**

用户取消和 CANCELLED 状态的超时完全不发送通知，但某些场景下可能需要告警。可以考虑增加配置项。

**问题3：on_pipeline_run_failure 直接取消块，未区分是否需要取消**

该函数总是调用 `cancel_block_runs_and_jobs()`，即使在"全部块结束后失败"场景下没有需要取消的块，也会做一次遍历。虽然影响不大，但逻辑上可以更清晰。

---

## 10. 关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 数据模型 | `mage_ai/orchestration/db/models/schedules.py` | PipelineRun / BlockRun 模型定义，状态枚举，complete() 方法 |
| 调度器 | `mage_ai/orchestration/pipeline_scheduler_original.py` | 管道调度、状态管理、超时检查、各种终止路径 |
| 调度器（平台版） | `mage_ai/orchestration/pipeline_scheduler_project_platform.py` | 项目平台版调度器 |
| Streaming 执行器 | `mage_ai/data_preparation/executors/streaming_pipeline_executor.py` | Streaming 管道状态更新（独立路径，始终设置 completed_at） |
| 日志管理 | `mage_ai/data_preparation/logging/logger_manager.py` | 日志存储、轮转、清理 |
| 日志工厂 | `mage_ai/data_preparation/logging/logger_manager_factory.py` | 日志管理器工厂 |
| 结构化日志 | `mage_ai/data_preparation/logging/logger.py` | DictLogger 实现 |
| 指标计算 | `mage_ai/orchestration/metrics/pipeline_run.py` | 管道运行指标计算 |
| 状态检查 | `mage_ai/orchestration/run_status_checker.py` | 运行状态检查（传感器用） |
| 前端日志 | `mage_ai/frontend/utils/models/log.ts` | 前端日志解析与格式化 |

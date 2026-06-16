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

| 源状态 | 目标状态 | 触发位置 | 触发条件 | completed_at | 失败通知 |
|--------|---------|---------|---------|-------------|---------|
| INITIAL | RUNNING | `PipelineScheduler.start()` L197-200 | 成功初始化块运行，获取分布式锁 | - | - |
| RUNNING | COMPLETED | `schedule()` L262 `complete()` | 所有块成功完成 | ✅ 设置 | ✅ 成功通知 |
| RUNNING | FAILED | `schedule()` L251-253 | 所有块完成但存在失败块 | ✅ 设置 | ✅ 发送 |
| RUNNING | FAILED | `schedule()` L309 | 块失败且 `allow_blocks_to_fail == False` | ❌ 未设置 | ✅ 发送 |
| RUNNING | FAILED | `schedule()` L306 | 管道超时（默认 FAILED） | ❌ 未设置 | ✅ 发送 |
| RUNNING | CANCELLED | `schedule()` L306 | 管道超时（配置为 CANCELLED） | ❌ 未设置 | ❌ 不发送 |
| RUNNING | CANCELLED | `stop_pipeline_run()` L1452 | 用户主动取消 | ❌ 未设置 | ❌ 不发送 |
| RUNNING | CANCELLED | `memory_usage_failure()` → `stop()` | 内存使用率 ≥ 95% | ❌ 未设置 | ✅ **发送（summary）** |
| RUNNING | FAILED | `start()` L187 | 初始化块运行时异常 | ❌ 未设置 | ✅ 发送（直接调用） |
| RUNNING | FAILED/其他 | `streaming_pipeline_executor.py` L288-289 | Streaming 管道结束 | ✅ **始终设置** | FAILED 时发送 |

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

## 4. 取消与失败路径详细对比

### 4.1 终止路径总览（七种场景）

| 终止场景 | 最终状态 | completed_at | 失败通知 | 调用 on_pipeline_run_failure | 调用 cancel_block_runs_and_jobs | 代码入口 |
|---------|---------|-------------|---------|------------------------------|--------------------------------|---------|
| **正常完成** | COMPLETED | ✅ 设置 | ✅ 成功通知 | ❌ | ❌ | `schedule()` L262 `complete()` |
| **全部块结束后失败** | FAILED | ✅ 设置 | ✅ 发送 | ✅ (status=FAILED) | ✅（无实际效果） | `schedule()` L251 |
| **运行中块失败** | FAILED | ❌ 未设置 | ✅ 发送 | ✅ (status=FAILED) | ✅ | `schedule()` L309 |
| **管道超时 (默认 FAILED)** | FAILED | ❌ 未设置 | ✅ 发送 | ✅ (status=FAILED) | ✅ | `schedule()` L306 |
| **管道超时 (配置为 CANCELLED)** | CANCELLED | ❌ 未设置 | ❌ **不发送** | ✅ (status=CANCELLED) | ✅ | `schedule()` L306 |
| **用户主动取消** | CANCELLED | ❌ 未设置 | ❌ **不发送** | ❌ **不调用** | ✅ | API → `stop()` → `stop_pipeline_run()` |
| **内存超限取消** | CANCELLED | ❌ 未设置 | ✅ **发送**（summary） | ❌ **不调用** | ✅（通过 stop()） | `memory_usage_failure()` → `stop()` |
| **start() 初始化失败** | FAILED | ❌ 未设置 | ✅ 发送 | ❌ 不调用 | ❌ 不调用 | `start()` L187 |
| **Streaming 管道任意终止** | FAILED/其他 | ✅ **始终设置** | ✅ FAILED 时发送 | ❌ 不调用 | ❌ 不调用 | `streaming_pipeline_executor.py` L288 |

---

### 4.2 取消分支的三种不同路径

#### 路径 A：用户主动取消

**触发时机**：用户通过 API 或 UI 点击取消按钮

**调用链路**：
```
API: cancel_pipeline_runs()  [mage_ai/api/resources/PipelineResource.py L780]
  └─ PipelineScheduler.stop()  [L206]
       └─ stop_pipeline_run(pipeline_run, pipeline)  [L1426]
            ├─ 前置状态检查（仅 INITIAL/RUNNING）
            ├─ 更新管道状态为 CANCELLED（不设置 completed_at）
            ├─ 记录使用统计
            └─ cancel_block_runs_and_jobs()  [L1460]
                 ├─ 收集 INITIAL/QUEUED/RUNNING 状态的块
                 ├─ 批量更新块为 CANCELLED（不设置 completed_at）
                 ├─ 终止相关作业
                 └─ 入队取消清理作业（on_pipeline_run_cancelled）
```

**关键特征**：
- 状态：CANCELLED
- **不调用 `on_pipeline_run_failure()`**
- **不发送失败通知**
- **不设置 `completed_at`**
- 块取消：批量 CANCELLED，不设置 completed_at

#### 路径 B：内存超限取消

**触发时机**：心跳检测发现系统内存使用率 ≥ 95%（仅 LOCAL_PYTHON 执行器）

**调用链路**：
```
schedule() → __run_heartbeat()
  └─ memory_usage >= MEMORY_USAGE_MAXIMUM (0.95)
       └─ memory_usage_failure(tags=tags)  [L490]
            ├─ 记录内存超限日志
            ├─ self.stop()  → 同用户取消路径
            │      └─ stop_pipeline_run()
            │           ├─ 状态 → CANCELLED
            │           └─ cancel_block_runs_and_jobs()
            ├─ ✅ 发送失败通知（summary=内存超限消息）  [L501]
            │   直接调用 notification_sender.send_pipeline_run_failure_message(summary=msg)
            └─ INTEGRATION 类型：计算管道运行指标
```

**关键特征**：
- 状态：CANCELLED（通过 `stop()` 设置）
- **不调用 `on_pipeline_run_failure()`**
- **会发送失败通知**（直接调用 `send_pipeline_run_failure_message`，参数为 `summary`）
- **不设置 `completed_at`**
- 是所有 CANCELLED 状态中**唯一会发送通知**的场景
- 通知内容是"内存使用率达到 95%"的 summary，没有 error 和 stacktrace

#### 路径 C：管道超时取消（timeout_status=CANCELLED）

**触发时机**：管道运行超时，且 `PipelineSchedule.timeout_status` 配置为 CANCELLED

**调用链路**：
```
schedule()
  └─ __check_pipeline_run_timeout() == True
       ├─ status = timeout_status or FAILED  → CANCELLED
       ├─ 更新管道状态为 CANCELLED（不设置 completed_at）  [L306]
       └─ on_pipeline_run_failure('Pipeline run timed out.', status=CANCELLED)  [L307]
            ├─ 记录使用统计
            ├─ ❌ status != FAILED → 不发送通知
            └─ cancel_block_runs_and_jobs()
                 └─ 同用户取消路径的块取消逻辑
```

**关键特征**：
- 状态：CANCELLED
- **调用 `on_pipeline_run_failure()`**，但传入 `status=CANCELLED`
- **不发送失败通知**（因为 status != FAILED，on_pipeline_run_failure 内部判断跳过）
- **不设置 `completed_at`**
- 是三种取消路径中**唯一经过 on_pipeline_run_failure** 的

---

### 4.3 三种取消路径的对比

| 维度 | 用户主动取消 | 内存超限取消 | 超时取消 (CANCELLED) |
|------|-------------|-------------|---------------------|
| **触发方式** | API/UI 调用 | 心跳检测内存 ≥ 95% | 调度循环检测超时 |
| **最终状态** | CANCELLED | CANCELLED | CANCELLED |
| **completed_at** | ❌ 未设置 | ❌ 未设置 | ❌ 未设置 |
| **失败通知** | ❌ 不发送 | ✅ **发送**（summary） | ❌ 不发送 |
| **调用 on_pipeline_run_failure** | ❌ 不调用 | ❌ 不调用 | ✅ 调用（status=CANCELLED） |
| **调用 stop_pipeline_run** | ✅ 是 | ✅ 是（通过 stop()） | ❌ 否（直接更新状态） |
| **取消块的方式** | cancel_block_runs_and_jobs | cancel_block_runs_and_jobs | cancel_block_runs_and_jobs |
| **通知发送方式** | - | 直接 send_pipeline_run_failure_message(summary=...) | - |
| **入队取消清理作业** | ✅ 是 | ✅ 是 | ✅ 是 |
| **记录使用统计** | ✅ 是（stop_pipeline_run 内） | ✅ 是（stop_pipeline_run 内） | ✅ 是（on_pipeline_run_failure 内） |

---

### 4.4 失败路径对比

#### 路径一：运行中块失败（allow_blocks_to_fail=False）

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

#### 路径二：全部块结束后发现失败

**触发时机**：所有块都已到达终态，但其中存在 FAILED 状态的块。常见于 `allow_blocks_to_fail=True` 的场景。

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

        # 5. 调用 on_pipeline_run_failure
        self.on_pipeline_run_failure(error_msg)
```

#### 路径三：管道超时（默认 FAILED）

**代码执行顺序**（`schedule()` L302-L307）：

```python
elif self.__check_pipeline_run_timeout():
    status = (
        self.pipeline_schedule.timeout_status or PipelineRun.PipelineRunStatus.FAILED
    )
    # 更新管道状态 —— 注意：没有设置 completed_at
    self.pipeline_run.update(status=status)
    # 调用 on_pipeline_run_failure，传入最终状态
    self.on_pipeline_run_failure('Pipeline run timed out.', status=status)
```

---

### 4.5 `on_pipeline_run_failure()` 的执行逻辑

**位置**：`pipeline_scheduler_original.py` L341-L374

```python
def on_pipeline_run_failure(self, error_msg, status=PipelineRun.PipelineRunStatus.FAILED):
    # 1. 记录使用统计
    UsageStatisticLogger().pipeline_run_ended_sync(self.pipeline_run)

    # 2. 仅当 status == FAILED 时发送失败通知
    if status == PipelineRun.PipelineRunStatus.FAILED:
        # 收集失败块的错误堆栈（最多 50 行，超出截断）
        failed_block_runs = self.pipeline_run.failed_block_runs
        stacktrace = None
        for br in failed_block_runs:
            if br.metrics:
                message = br.metrics.get('error', {}).get('message')
                if message:
                    # 截断到 50 行
                    ...
                    stacktrace = f'Error for block {br.block_uuid}:\n{message}'
                    break

        self.notification_sender.send_pipeline_run_failure_message(
            pipeline=self.pipeline,
            pipeline_run=self.pipeline_run,
            error=error_msg,
            stacktrace=stacktrace,
        )

    # 3. 取消所有进行中的块运行 + 终止作业
    cancel_block_runs_and_jobs(self.pipeline_run, self.pipeline)
```

**关键逻辑**：
- `status == FAILED` 时才发送通知，CANCELLED 不发送
- 总是调用 `cancel_block_runs_and_jobs()`，无论 status 是什么
- error 参数是简要错误消息，stacktrace 是从第一个失败块的 metrics.error.message 中提取

---

### 4.6 块取消与状态更新详解

**函数**：`cancel_block_runs_and_jobs(pipeline_run, pipeline)`

**位置**：`pipeline_scheduler_original.py` L1460-L1519

```python
def cancel_block_runs_and_jobs(pipeline_run, pipeline):
    # 1. 收集需要取消的块
    block_runs_to_cancel = []
    running_blocks = []
    for b in pipeline_run.block_runs:
        if b.status in [INITIAL, QUEUED, RUNNING]:
            block_runs_to_cancel.append(b)
        if b.status == RUNNING:
            running_blocks.append(b)
    cancelled_block_run_ids = [b.id for b in block_runs_to_cancel]

    # 2. 批量更新块状态为 CANCELLED —— 注意：不设置 completed_at
    BlockRun.batch_update_status(
        cancelled_block_run_ids,
        BlockRun.BlockRunStatus.CANCELLED,
    )

    # 3. 终止相关作业
    if pipeline and (pipeline.type in [INTEGRATION, STREAMING] or pipeline.run_pipeline_in_one_process):
        # 整体管道作业
        job_manager.kill_pipeline_run_job(pipeline_run.id)
        if pipeline.type == INTEGRATION:
            for stream in pipeline.streams():
                job_manager.kill_integration_stream_job(...)
        if pipeline_run.executor_type == K8S:
            ExecutorFactory.get_pipeline_executor(...).cancel(...)
    else:
        # 逐个块作业
        for b in running_blocks:
            job_manager.kill_block_run_job(b.id)

    # 4. 入队取消清理作业（执行 on_cancelled 回调）
    GenericJob.enqueue_cancel_pipeline_run(pipeline_run.id, cancelled_block_run_ids)
```

**块取消的状态更新细节**：
- 只取消状态为 INITIAL、QUEUED、RUNNING 的块
- 已经是 COMPLETED、FAILED、UPSTREAM_FAILED、CONDITION_FAILED 的块不受影响
- **批量更新为 CANCELLED 时不设置 completed_at**
- 作业终止只针对 RUNNING 状态的块（running_blocks）

---

### 4.7 取消后清理：`on_pipeline_run_cancelled()`

**位置**：`pipeline_scheduler_original.py` L936-L1008

这是一个异步执行的清理作业，在取消操作入队后执行。

```python
def on_pipeline_run_cancelled(job_id, pipeline_run_id, cancelled_block_run_ids):
    # 1. 作业状态检查和标记
    job = GenericJob.query.get(job_id)
    if job.status not in [INITIAL, QUEUED, RUNNING]:
        return
    job.mark_running()

    # 2. 参数校验
    if not pipeline_run_id:
        job.mark_failed(...)
        return
    if not cancelled_block_run_ids:
        job.mark_failed(...)
        return

    try:
        # 3. 验证管道运行状态（必须是 CANCELLED）
        pipeline_run = PipelineRun.query.get(pipeline_run_id)
        if pipeline_run.status != PipelineRun.PipelineRunStatus.CANCELLED:
            job.mark_failed(...)
            return

        # 4. 对每个被取消的块执行 on_cancelled 回调
        for block_run_id in cancelled_block_run_ids:
            block_run = BlockRun.query.get(block_run_id)
            if not block_run or block_run.status != BlockRun.BlockRunStatus.CANCELLED:
                continue
            ExecutorFactory.get_block_executor(...).execute_callback(
                callback='on_cancelled',
                global_vars=pipeline_run.get_variables(),
                pipeline_run=pipeline_run,
                block_run_id=block_run_id,
            )
    except Exception as e:
        job.mark_failed(...)
        return
    job.mark_completed()
```

**作用**：执行块的 `on_cancelled` 回调，用于取消后的清理工作。

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

### 8.1 最容易混淆的 8 个结论

| 混淆点 | 常见错误理解 | 正确结论 |
|--------|-------------|---------|
| **CANCELLED 不发通知** | 所有 CANCELLED 状态都不发送通知 | ❌ 不对。**内存超限取消是 CANCELLED 状态，但会发送失败通知**（直接调用 send_pipeline_run_failure_message） |
| **取消都走同一路径** | 用户取消、超时取消、内存超限取消走相同代码路径 | ❌ 不对。三条路径完全不同：用户取消走 `stop_pipeline_run()`，超时取消走 `on_pipeline_run_failure(status=CANCELLED)`，内存超限走 `memory_usage_failure()` → `stop()` + 单独发通知 |
| **completed_at 设置** | FAILED 状态都有 completed_at | ❌ 不对。**全部块结束后失败才有 completed_at，运行中块失败没有** |
| **on_pipeline_run_failure 只处理失败** | 该函数只处理 FAILED 状态 | ❌ 不对。它也处理 CANCELLED（超时取消时），只是 CANCELLED 时跳过通知发送 |
| **取消就是用户操作** | CANCELLED 状态都是用户主动取消的 | ❌ 不对。CANCELLED 可能来自：用户取消、内存超限、超时配置为 CANCELLED |
| **块取消只在用户取消时** | 只有用户取消才会取消块 | ❌ 不对。运行中块失败、超时（FAILED/CANCELLED）、内存超限、用户取消**都会调用 `cancel_block_runs_and_jobs()`** |
| **失败通知都走 on_pipeline_run_failure** | 发送失败通知都经过该函数 | ❌ 不对。内存超限取消、start() 初始化失败、Streaming 管道失败都是直接调用 `send_pipeline_run_failure_message`，不经过 on_pipeline_run_failure |
| **失败都有 error 和 stacktrace** | 失败通知都包含 error 和 stacktrace | ❌ 不对。内存超限取消的通知只有 summary，没有 error 和 stacktrace |

---

### 8.2 三种取消路径的对比速查

| 维度 | 用户主动取消 | 内存超限取消 | 超时取消 (CANCELLED) |
|------|-------------|-------------|---------------------|
| **入口函数** | API → `cancel_pipeline_runs()` | `memory_usage_failure()` | `schedule()` 超时检测 |
| **是否调用 stop_pipeline_run** | ✅ 是 | ✅ 是（通过 stop()） | ❌ 否 |
| **是否调用 on_pipeline_run_failure** | ❌ 否 | ❌ 否 | ✅ 是（status=CANCELLED） |
| **是否发送失败通知** | ❌ 不发送 | ✅ 发送（summary） | ❌ 不发送 |
| **通知调用方式** | - | 直接 `send_pipeline_run_failure_message(summary=...)` | - |
| **completed_at** | ❌ 未设置 | ❌ 未设置 | ❌ 未设置 |
| **块取消方式** | `cancel_block_runs_and_jobs()` | `cancel_block_runs_and_jobs()` | `cancel_block_runs_and_jobs()` |
| **最终管道状态** | CANCELLED | CANCELLED | CANCELLED |
| **是否入队取消清理作业** | ✅ 是 | ✅ 是 | ✅ 是 |
| **是否记录使用统计** | ✅ 是（stop_pipeline_run 内） | ✅ 是（stop_pipeline_run 内） | ✅ 是（on_pipeline_run_failure 内） |

---

### 8.3 completed_at 设置情况速查（PipelineRun）

| 终止场景 | 最终状态 | completed_at | 代码位置 |
|---------|---------|-------------|---------|
| 正常完成 | COMPLETED | ✅ 设置 | `PipelineRun.complete()` L1408 |
| 全部块结束后失败 | FAILED | ✅ 设置 | `schedule()` L253 |
| 运行中块失败 | FAILED | ❌ 未设置 | `schedule()` L309 |
| 管道超时 (默认) | FAILED | ❌ 未设置 | `schedule()` L306 |
| 管道超时 (配置 CANCELLED) | CANCELLED | ❌ 未设置 | `schedule()` L306 |
| 用户主动取消 | CANCELLED | ❌ 未设置 | `stop_pipeline_run()` L1452 |
| 内存超限取消 | CANCELLED | ❌ 未设置 | `stop_pipeline_run()` L1452 |
| start() 初始化失败 | FAILED | ❌ 未设置 | `start()` L187 |
| Streaming 管道任意终止 | FAILED / 其他 | ✅ **始终设置** | `streaming_pipeline_executor.py` L289 |

**关键发现**：
- 非 Streaming 管道中，只有「所有块都已到达终态」的场景才设置 completed_at（正常完成、全部块结束后失败）
- 运行中被终止的场景（运行中块失败、超时、取消）都不设置 completed_at
- Streaming 管道走独立路径，始终设置 completed_at

---

### 8.4 BlockRun 的 completed_at 设置情况

| 块终止场景 | 最终状态 | completed_at | 代码位置 |
|-----------|---------|-------------|---------|
| 正常完成 | COMPLETED | ✅ 设置 | `on_block_complete()` L392 |
| 执行失败 | FAILED | ❌ **未设置** | `on_block_failure()` L450-453 |
| 被批量取消 | CANCELLED | ❌ **未设置** | `BlockRun.batch_update_status()` |
| 上游失败 | UPSTREAM_FAILED | ❌ 未设置 | `update_block_run_statuses()` |
| 条件失败 | CONDITION_FAILED | ❌ 未设置 | `update_block_run_statuses()` |

**注意**：BlockRun 的 FAILED 和 CANCELLED 状态都**不设置 completed_at**，只有 COMPLETED 才设置。这与 PipelineRun 的行为一致——被中断/失败的都不记录结束时间。

---

### 8.5 失败通知发送路径汇总

| 场景 | 状态 | 是否发送 | 调用路径 | 通知内容 |
|------|------|---------|---------|---------|
| 全部块结束后失败 | FAILED | ✅ 发送 | `on_pipeline_run_failure()` → `send_pipeline_run_failure_message` | error + stacktrace |
| 运行中块失败 | FAILED | ✅ 发送 | `on_pipeline_run_failure()` → `send_pipeline_run_failure_message` | error + stacktrace |
| 管道超时 (FAILED) | FAILED | ✅ 发送 | `on_pipeline_run_failure()` → `send_pipeline_run_failure_message` | error + stacktrace |
| 管道超时 (CANCELLED) | CANCELLED | ❌ 不发送 | - | - |
| 用户取消 | CANCELLED | ❌ 不发送 | - | - |
| 内存超限取消 | CANCELLED | ✅ 发送 | 直接 `send_pipeline_run_failure_message(summary=...)` | summary（无 error/stacktrace） |
| start() 初始化失败 | FAILED | ✅ 发送 | 直接 `send_pipeline_run_failure_message(error=...)` | error（无 stacktrace） |
| Streaming 管道失败 | FAILED | ✅ 发送 | 直接 `send_pipeline_run_failure_message` | error + stacktrace |
| 正常完成 | COMPLETED | ✅ 成功通知 | `send_pipeline_run_success_message` | 成功消息 |

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
8. **取消后回调机制**：通过 `on_pipeline_run_cancelled` 异步作业执行块的 `on_cancelled` 回调

### 9.2 可改进点

**问题1：completed_at 设置不一致**

多个终止路径不设置 `completed_at`，导致：
- 无法准确计算运行时长（只能用 started_at + 当前时间估算）
- 数据统计和分析不完整
- Streaming 与非 Streaming 管道行为不一致

涉及的不设置位置：
- `schedule()` L309（运行中块失败）
- `schedule()` L306（管道超时，FAILED 和 CANCELLED）
- `stop_pipeline_run()` L1452（用户取消、内存超限取消）
- `start()` L187（初始化失败）
- `BlockRun.batch_update_status()`（块取消）
- `on_block_failure()`（块失败）

建议：在所有终止路径统一设置 `completed_at`，保持数据一致性。

**问题2：取消路径不统一，逻辑分散**

三种取消路径（用户取消、内存超限、超时取消）走的代码分支完全不同，通知行为也不一致：
- 用户取消：不发通知
- 内存超限：发通知（summary）
- 超时取消（CANCELLED）：不发通知

建议：统一取消处理逻辑，增加配置项控制是否发送取消通知。

**问题3：on_pipeline_run_failure 命名有歧义**

该函数名暗示只处理失败，但实际上它也处理 CANCELLED 状态（超时取消时）。只是 CANCELLED 时跳过通知发送，但仍然会执行 `cancel_block_runs_and_jobs()`。更准确的命名可能是 `on_pipeline_run_terminated()` 或 `handle_pipeline_run_end()`。

**问题4：BlockRun 的 FAILED 状态没有 completed_at**

块失败时不设置 completed_at，但块的运行时长是可以计算的（started_at 到失败时间）。这使得无法准确统计失败块的执行时长。

**问题5：内存超限取消的通知内容不完整**

内存超限取消发送的失败通知只有 summary，没有 error 和 stacktrace。如果用户习惯在失败通知中查找错误信息，可能会困惑。建议统一通知格式，或明确标记为"取消通知"而非"失败通知"。

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

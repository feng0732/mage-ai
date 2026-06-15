# Pipeline Run 日志代码分析

## 1. 核心抽象

### 1.1 数据模型层

#### PipelineRun 模型
[PipelineRun](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L768-L912) 是管道运行的核心数据模型，代表一次管道执行实例。

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
    metrics = Column(JSON)
    backfill_id = Column(Integer, ForeignKey('backfill.id'), index=True)
    executor_type = Column(Enum(ExecutorType), default=ExecutorType.LOCAL_PYTHON)
```

**核心属性和方法**：
- `execution_partition` [L813-L826](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L813-L826)：日志分区标识，格式为 `{schedule_id}/{execution_date}`
- `logs` [L873-L892](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L873-L892)：获取管道日志和调度器日志
- `all_blocks_completed()` [L1522-L1532](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1522-L1532)：检查所有块是否完成
- `any_blocks_failed()` [L1519-L1520](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1519-L1520)：检查是否有块失败
- `complete()` [L1406-L1416](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1406-L1416)：标记管道运行为完成状态
- `create_block_runs()` [L1443-L1517](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1443-L1517)：创建所有块运行实例

#### BlockRun 模型
[BlockRun](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1663-L1712) 表示管道中单个块的执行实例。

```python
class BlockRun(BlockRunProjectPlatformMixin, BaseModel):
    class BlockRunStatus(StrEnum):
        INITIAL = 'initial'           # 初始状态
        QUEUED = 'queued'             # 已排队
        RUNNING = 'running'           # 运行中
        COMPLETED = 'completed'       # 已完成
        FAILED = 'failed'             # 失败
        CANCELLED = 'cancelled'       # 已取消
        UPSTREAM_FAILED = 'upstream_failed'    # 上游失败
        CONDITION_FAILED = 'condition_failed'  # 条件失败

    pipeline_run_id = Column(Integer, ForeignKey('pipeline_run.id'), index=True)
    block_uuid = Column(String(255))
    status = Column(Enum(BlockRunStatus), default=BlockRunStatus.INITIAL)
    started_at = Column(DateTime(timezone=True))
    completed_at = Column(DateTime(timezone=True))
    metrics = Column(JSON)
```

**核心属性和方法**：
- `logs` [L1684-L1691](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1684-L1691)：获取块运行日志
- `batch_update_status()` [L1706-L1711](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1706-L1711)：批量更新块运行状态

### 1.2 日志管理层

#### LoggerManager
[LoggerManager](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger_manager.py#L23-L195) 是日志管理的核心类，负责日志的创建、存储和检索。

**核心功能**：
- 支持文件日志和流日志两种模式
- 使用 `RotatingFileHandler` 实现日志轮转（最大 20MB，保留 10 个备份）
- 按 `pipeline_uuid/partition/block_uuid` 分层存储日志
- 支持日志保留期配置，自动清理旧日志

```python
class LoggerManager:
    def __init__(
        self,
        repo_path: str = None,
        logs_dir: str = None,
        pipeline_uuid: str = None,
        block_uuid: str = None,
        filename: str = None,
        partition: str = None,
        repo_config: RepoConfig = None,
        subpartition: str = None,
    ):
```

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
[LoggerManagerFactory](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger_manager_factory.py#L6-L22) 是工厂类，根据配置创建不同类型的日志管理器。

```python
class LoggerManagerFactory:
    @classmethod
    def get_logger_manager(self, repo_config: RepoConfig = None, **kwargs):
        if repo_config is not None and repo_config.logging_config is not None:
            logger_type = repo_config.logging_config.get('type')
            if logger_type == LoggerType.S3:
                return S3LoggerManager(repo_config=repo_config, **kwargs)
            elif logger_type == LoggerType.GCS:
                return GCSLoggerManager(repo_config=repo_config, **kwargs)
        return LoggerManager(repo_config=repo_config, **kwargs)
```

#### DictLogger
[DictLogger](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger.py#L13-L92) 提供结构化日志记录，将日志消息编码为 JSON 格式。

```python
class DictLogger():
    def __init__(self, logger: logging.Logger, logging_tags: Dict = None):
        self.logger = logger
        self.logging_tags = logging_tags or dict()

    def __send_message(self, method_name, message, error=None, log_level=None, **kwargs):
        now = datetime.utcnow()
        data = dict(
            level=logging.getLevelName(log_level) if log_level else method_name.upper(),
            message=message,
            timestamp=now.timestamp(),
            uuid=uuid.uuid4().hex,
        )
        if error:
            data['error'] = traceback.format_exc()
            data['error_stack'] = traceback.format_stack()
            data['error_stacktrace'] = str(error)

        msg = simplejson.dumps(
            merge_dict(self.logging_tags or dict(), merge_dict(kwargs, data)),
            default=encode_complex,
            ignore_nan=True,
        )
```

**日志格式**：
```json
{
  "level": "INFO",
  "message": "Execute PipelineRun 123",
  "timestamp": 1718562466.123,
  "uuid": "abc123...",
  "pipeline_run_id": 123,
  "pipeline_uuid": "my_pipeline",
  "hostname": "server-01"
}
```

### 1.3 调度层

#### PipelineScheduler
[PipelineScheduler](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L78-L131) 是管道运行的调度核心，负责协调管道和块的执行。

```python
class PipelineScheduler:
    def __init__(self, pipeline_run: PipelineRun) -> None:
        self.pipeline_run = pipeline_run
        self.pipeline_schedule = pipeline_run.pipeline_schedule
        self.pipeline = get_pipeline_from_platform(...)

        # 初始化日志管理器
        self.logger_manager = LoggerManagerFactory.get_logger_manager(
            pipeline_uuid=self.pipeline.uuid,
            filename='scheduler.log',
            partition=self.pipeline_run.execution_partition,
            repo_config=self.pipeline.repo_config,
        )
        self.logger = DictLogger(self.logger_manager.logger)

        # 初始化通知发送器
        self.notification_sender = NotificationSender(...)

        # 并发配置
        self.concurrency_config = ConcurrencyConfig.load(...)
```

---

## 2. 状态切换

### 2.1 PipelineRun 状态机

```
INITIAL
   │
   ▼
RUNNING ─────────┐
   │             │
   ├────────────► COMPLETED (所有块成功完成)
   │             │
   ├────────────► FAILED    (块失败且不允许失败)
   │             │
   └────────────► CANCELLED (用户取消或超时)
```

**状态切换触发点**：

1. **INITIAL → RUNNING**
   - 位置：[PipelineScheduler.start()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L197-L200)
   - 条件：成功初始化块运行，获取分布式锁
   - 操作：设置 `started_at`，更新状态为 RUNNING

2. **RUNNING → COMPLETED**
   - 位置：[PipelineScheduler.schedule()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L240-L267)
   - 条件：`all_blocks_completed() == True` 且 `any_blocks_failed() == False`
   - 操作：调用 `pipeline_run.complete()`，设置 `completed_at`，发送成功通知

3. **RUNNING → FAILED**
   - 位置1：块失败且 `allow_blocks_to_fail == False` [L308-L331](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L308-L331)
   - 位置2：超时 [L302-L307](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L302-L307)
   - 操作：更新状态，取消未完成块，发送失败通知

4. **RUNNING → CANCELLED**
   - 位置：[stop_pipeline_run()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1426-L1457)
   - 条件：用户主动取消，或内存超限
   - 操作：更新状态，取消所有运行中的块，终止相关作业

### 2.2 BlockRun 状态机

```
INITIAL
   │
   ▼
QUEUED ────────────────────┐
   │                       │
   ▼                       │
RUNNING ──────────┐        │
   │              │        │
   ├────────────► COMPLETED      (成功完成)
   │              │        │
   ├────────────► FAILED         (执行失败)
   │              │        │
   ├────────────► CANCELLED      (被取消)
   │              │        │
   └────────────► UPSTREAM_FAILED (上游失败)
                  │        │
                  └────────┴─── CONDITION_FAILED (条件不满足)
```

**状态切换触发点**：

1. **INITIAL → QUEUED → RUNNING**
   - 位置：[execute_block_run()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1279-L1283)
   - 条件：块可执行，所有上游依赖完成

2. **RUNNING → COMPLETED**
   - 位置：[on_block_complete()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L377-L410)
   - 操作：设置 `completed_at`，更新 metrics，触发下一轮调度

3. **RUNNING → FAILED**
   - 位置：[on_block_failure()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L443-L488)
   - 操作：记录错误信息到 metrics，更新状态，根据配置决定是否终止管道

4. **批量状态更新**
   - 位置：[cancel_block_runs_and_jobs()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1490-L1493)
   - 使用 `BlockRun.batch_update_status()` 批量更新状态为 CANCELLED

### 2.3 状态检查机制

[run_status_checker.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/run_status_checker.py) 提供状态检查功能，主要用于传感器（Sensor）场景。

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

---

## 3. 边界处理

### 3.1 并发控制

#### 分布式锁
[PipelineScheduler.start()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L148-L195) 使用分布式锁防止并发启动：

```python
lock_key = f'pipeline_run_{self.pipeline_run.id}'
if not lock.try_acquire_lock(lock_key):
    return
try:
    # 初始化块运行...
finally:
    lock.release_lock(lock_key)
```

#### 并发配置
`ConcurrencyConfig` 控制并发执行策略，包括：
- 最大并发块数
- 达到限制时的行为（排队或拒绝）

### 3.2 超时处理

#### 管道运行超时
[__check_pipeline_run_timeout()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L528-L555)：

```python
def __check_pipeline_run_timeout(self) -> bool:
    try:
        pipeline_run_timeout = self.pipeline_schedule.timeout
        if self.pipeline_run.started_at and pipeline_run_timeout:
            time_difference = (
                datetime.now(tz=pytz.UTC).timestamp()
                - self.pipeline_run.started_at.timestamp()
            )
            if time_difference > int(pipeline_run_timeout):
                return True
    except Exception:
        pass
    return False
```

#### 块运行超时
[__check_block_run_timeout()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L557-L589)：
- 检查每个运行中的块是否超过其配置的超时时间
- 超时的块会被标记为 FAILED

### 3.3 错误处理

#### 块失败策略
[on_block_failure()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L443-L488)：
- `allow_blocks_to_fail == True`：块失败不影响管道继续运行
- `allow_blocks_to_fail == False`：块失败导致整个管道失败
- 对于 INTEGRATION 类型管道，块失败会终止所有流

#### 重试机制
块状态更新使用 `@retry` 装饰器，确保数据库操作的可靠性：

```python
@retry(retries=2, delay=5)
def update_status():
    block_run.update(
        status=BlockRun.BlockRunStatus.COMPLETED,
        completed_at=datetime.now(tz=pytz.UTC),
        metrics=metrics_prev,
    )
```

### 3.4 取消逻辑

[stop_pipeline_run()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1426-L1457)：

1. **前置检查**：仅允许取消 INITIAL 或 RUNNING 状态的管道
2. **状态更新**：更新管道状态为 CANCELLED
3. **块取消**：批量更新所有 INITIAL/QUEUED/RUNNING 状态的块为 CANCELLED
4. **作业终止**：
   - 对于 INTEGRATION/STREAMING 管道，终止整个管道作业和所有流作业
   - 对于普通管道，终止单个块作业
   - 对于 K8S 执行器，调用 executor.cancel()

### 3.5 内存保护

[memory_usage_failure()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L490-L513)：
- 当内存使用率超过 95% 时，自动停止管道运行
- 发送内存超限通知

### 3.6 日志边界

#### 日志轮转
`RotatingFileHandler` 配置：
- 最大文件大小：20MB
- 备份文件数：10个

#### 日志清理
[delete_old_logs()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger_manager.py#L144-L195)：
- 根据 `retention_period` 配置自动清理过期日志
- Streaming 管道日志跳过清理
- 按 partition（执行日期）进行清理判断

---

## 4. 关键交互流程

### 4.1 管道启动流程

```
1. PipelineScheduler.start()
   ├─ 检查状态，获取分布式锁
   ├─ 初始化 BlockRun（create_block_runs）
   ├─ 更新 PipelineRun 状态为 RUNNING
   └─ 调用 schedule() 开始调度

2. PipelineScheduler.schedule()
   ├─ 检查所有块是否完成
   │  ├─ 是 → 标记 COMPLETED/FAILED
   │  └─ 否 → 继续调度
   ├─ 检查超时
   ├─ 检查块失败
   └─ 调度可执行块
```

### 4.2 块执行流程

```
1. execute_block_run(pipeline_run_id, block_run_id)
   ├─ 检查 PipelineRun 状态（必须为 RUNNING）
   ├─ 检查 BlockRun 状态（INITIAL/QUEUED/RUNNING）
   ├─ 更新 BlockRun 状态为 RUNNING
   ├─ 创建 BlockExecutor
   └─ 执行块
      ├─ 成功 → on_block_complete()
      │   ├─ 更新状态为 COMPLETED
      │   ├─ 更新 metrics
      │   └─ 触发 schedule()
      └─ 失败 → on_block_failure()
          ├─ 记录错误到 metrics
          ├─ 更新状态为 FAILED
          └─ 根据配置决定是否终止管道
```

### 4.3 日志收集流程

```
DictLogger.info(message, **tags)
    │
    ▼
__send_message()
    ├─ 添加 timestamp, uuid, level
    ├─ 合并 logging_tags 和 kwargs
    ├─ error 时添加 stack trace
    ├─ JSON 序列化
    └─ 调用底层 logger 输出
        │
        ▼
LoggerManager 的 Handler
    ├─ 写入文件（RotatingFileHandler）
    └─ 或写入流（StreamHandler）
```

---

## 5. 设计亮点与可改进点

### 5.1 设计亮点

1. **分层日志架构**：通过工厂模式支持多种日志存储后端（本地、S3、GCS）
2. **结构化日志**：使用 JSON 格式便于日志分析和检索
3. **分布式锁**：防止并发调度冲突
4. **状态完整性**：覆盖了各种失败场景的状态处理
5. **可观测性**：metrics 字段支持丰富的运行指标收集

### 5.2 代码优化建议

**问题**：[stop_pipeline_run()](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1426-L1457) 中状态更新后未设置 `completed_at`

```python
# 当前代码
def stop_pipeline_run(pipeline_run: PipelineRun, ...) -> None:
    if pipeline_run.status not in [INITIAL, RUNNING]:
        return
    pipeline_run.update(status=status)  # 缺少 completed_at
    ...
```

**建议优化**：

```python
def stop_pipeline_run(pipeline_run: PipelineRun, ...) -> None:
    if pipeline_run.status not in [INITIAL, RUNNING]:
        return
    pipeline_run.update(
        status=status,
        completed_at=datetime.now(tz=pytz.UTC),  # 添加 completed_at
    )
    ...
```

这样可以确保被取消的管道运行也有明确的结束时间。

---

## 6. 核心类关系图

```
PipelineRun
  │  (1)
  │  包含
  ▼
BlockRun (N)
  │
  ├─ status: BlockRunStatus
  ├─ metrics: JSON (错误信息、执行指标)
  └─ logs (通过 LoggerManager 获取)

LoggerManagerFactory
  │
  ├─ 创建 LoggerManager
  │   ├─ pipeline_uuid
  │   ├─ partition (execution_partition)
  │   ├─ block_uuid
  │   └─ filename (scheduler.log, pipeline.log)
  │
  ├─ S3LoggerManager (可选)
  └─ GCSLoggerManager (可选)

DictLogger
  └─ 包装 logging.Logger，输出结构化 JSON 日志

PipelineScheduler
  ├─ pipeline_run: PipelineRun
  ├─ logger: DictLogger
  ├─ logger_manager: LoggerManager
  └─ notification_sender: NotificationSender
```

---

## 7. 关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 数据模型 | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/db/models/schedules.py) | PipelineRun/BlockRun 模型定义 |
| 调度器 | [pipeline_scheduler_original.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py) | 管道调度、状态管理 |
| 日志管理 | [logger_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger_manager.py) | 日志存储、轮转、清理 |
| 日志工厂 | [logger_manager_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger_manager_factory.py) | 日志管理器工厂 |
| 结构化日志 | [logger.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/data_preparation/logging/logger.py) | DictLogger 实现 |
| 指标计算 | [pipeline_run.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/metrics/pipeline_run.py) | 运行指标计算 |
| 状态检查 | [run_status_checker.py](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/orchestration/run_status_checker.py) | 运行状态检查 |
| 前端日志 | [log.ts](file:///d:/fz/0601/solo-dogfeeding/code/322-mage-ai/mage_ai/frontend/utils/models/log.ts) | 前端日志解析 |

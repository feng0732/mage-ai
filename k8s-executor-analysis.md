# K8s 执行器代码链路深度分析

> 代码路径均使用仓库根目录的相对路径表示。

---

## 一、核心文件索引

| 模块 | 仓库相对路径 | 职责 |
|------|-------------|------|
| Pipeline 级 K8s 执行器 | `mage_ai/data_preparation/executors/k8s_pipeline_executor.py` | 封装 Pipeline 在 K8s 上的执行与取消 |
| Block 级 K8s 执行器 | `mage_ai/data_preparation/executors/k8s_block_executor.py` | 封装单个 Block 在 K8s 上的执行 |
| 执行器工厂 | `mage_ai/data_preparation/executors/executor_factory.py` | 根据 executor_type 分派到对应执行器 |
| K8s Job 管理器 | `mage_ai/services/k8s/job_manager.py` | K8s Job 的创建、轮询、删除等底层操作 |
| K8s 配置解析 | `mage_ai/services/k8s/config.py` | K8sExecutorConfig 数据类，解析 Pod/Container/Job 配置 |
| Mage 内部进程级 Job 管理器 | `mage_ai/orchestration/job_manager.py` | Mage 自身的 Job 排队与 kill 入口（非 K8s） |
| Mage 内部进程队列 | `mage_ai/orchestration/queue/process_queue.py` | ProcessQueue 实现，含 kill_job 的 SIGKILL 行为 |
| Pipeline 调度器 | `mage_ai/orchestration/pipeline_scheduler_original.py` | PipelineRun/BlockRun 的调度、状态推进、取消入口 |
| 状态模型 | `mage_ai/orchestration/db/models/schedules.py` | PipelineRun、BlockRun 数据模型及状态枚举 |
| CLI 入口（Pod 内部实际执行） | `mage_ai/cli/main.py` | `mage run` 命令，K8s Pod 启动后真正执行的入口 |
| Pipeline 执行器基类 | `mage_ai/data_preparation/executors/pipeline_executor.py` | 提供 `_run_commands()` 构建 Pod 内的 CLI 命令 |
| Block 执行器基类 | `mage_ai/data_preparation/executors/block_executor.py` | 提供 `_run_commands()`、`__update_block_run_status()` 及 on_complete/on_failure 回调调度 |

---

## 二、入口协作：从触发到 K8s Job 提交

> **边界定义**：入口协作阶段从 PipelineRun 被创建并标记为 RUNNING 开始，到 K8s Job 对象被成功提交到 K8s API Server 为止。此阶段不涉及 Pod 内部的实际业务执行。

### 2.1 调度起点：PipelineScheduler.start()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L130-L203

```python
def start(self, should_schedule: bool = True) -> bool:
    self.pipeline_run.create_block_runs()     # 1. 创建 BlockRun 记录
    self.pipeline_run.update(                  # 2. PipelineRun: INITIAL → RUNNING
        started_at=datetime.now(tz=pytz.UTC),
        status=PipelineRun.PipelineRunStatus.RUNNING,
    )
    if should_schedule:
        self.schedule()                        # 3. 进入调度分支
```

### 2.2 调度分支：两种执行粒度

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L212-L338

```
PipelineScheduler.schedule()
    │
    ├─► pipeline.run_pipeline_in_one_process == True
    │      └─► __schedule_pipeline()   「整 Pipeline 一个 K8s Job」
    │
    ├─► pipeline.type == STREAMING
    │      └─► __schedule_pipeline()   「整 Pipeline 一个 K8s Job」
    │
    ├─► pipeline.type == INTEGRATION
    │      └─► __schedule_integration_streams()  「集成管道特殊逻辑」
    │
    └─► 其他（默认批处理）
           └─► __schedule_blocks()     「每个 Block 一个 K8s Job」
```

### 2.3 路径 A：整 Pipeline 一个 K8s Job

#### 2.3.1 提交 Mage 内部进程级 Job：__schedule_pipeline()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L836-L869

```python
def __schedule_pipeline(self):
    job_manager.add_job(          # 注意：这是 Mage 内部 JobManager，不是 K8s JobManager
        JobType.PIPELINE_RUN,
        self.pipeline_run.id,
        run_pipeline,             # 回调函数：在 Mage 进程池线程中执行
        self.pipeline_run.id,
        self.pipeline_run.get_variables(),
        self.build_tags(),
    )
```

`job_manager` 是 `mage_ai/orchestration/job_manager.py` 中的单例，封装了进程队列 `ProcessQueue`。

#### 2.3.2 进程池执行函数：run_pipeline()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1336-L1362

```python
def run_pipeline(pipeline_run_id, variables, tags, allow_blocks_to_fail=False):
    pipeline_run = PipelineRun.query.get(pipeline_run_id)
    executor_type = ExecutorFactory.get_pipeline_executor_type(pipeline)
    pipeline_run.update(executor_type=executor_type)

    # K8S → K8sPipelineExecutor
    ExecutorFactory.get_pipeline_executor(
        pipeline,
        execution_partition=pipeline_run.execution_partition,
        executor_type=executor_type,
    ).execute(
        allow_blocks_to_fail=allow_blocks_to_fail,
        global_vars=variables,
        pipeline_run_id=pipeline_run_id,
        tags=tags,
    )
```

#### 2.3.3 执行器分派：ExecutorFactory

文件：`mage_ai/data_preparation/executors/executor_factory.py`，L39-L92

```python
elif executor_type == ExecutorType.K8S:
    from mage_ai.data_preparation.executors.k8s_pipeline_executor import K8sPipelineExecutor
    return K8sPipelineExecutor(pipeline, execution_partition=execution_partition)
```

#### 2.3.4 K8sPipelineExecutor.execute() → 提交 K8s Job

文件：`mage_ai/data_preparation/executors/k8s_pipeline_executor.py`，L44-L63

```python
def execute(self, pipeline_run_id=None, global_vars=None, **kwargs):
    job_manager = self.get_job_manager(      # 这才是 K8s 层面的 JobManager
        pipeline_run_id=pipeline_run_id, global_vars=global_vars, **kwargs)
    cmd = self._run_commands(                # 构建 Pod 内 CLI 命令
        global_vars=global_vars, pipeline_run_id=pipeline_run_id, **kwargs)
    job_manager.run_job(cmd, k8s_config=self.executor_config)   # 提交并阻塞等待
```

Pod 内执行命令由基类 `mage_ai/data_preparation/executors/pipeline_executor.py` L181-L215 构建：

```bash
/app/run_app.sh mage run <repo_path> <pipeline_uuid> \
  --executor-type local_python \
  [--execution-partition <partition>] \
  [--pipeline-run-id <id>]
```

> **关键事实**：K8s Job 内部仍然使用 `local_python` 执行器。K8s 只负责把执行环境搬到独立 Pod，Pod 内部执行的就是常规的本地 Pipeline 流程。

#### 2.3.5 Job 命名

文件：`mage_ai/data_preparation/executors/k8s_pipeline_executor.py`，L65-L89

```
{ MAGE_CLUSTER_UUID } - { job_name_prefix } - pipeline - { pipeline_run_id }
```

`job_name_prefix` 支持 `{trigger_name}` 模板，会替换为关联 Trigger 的名称。

### 2.4 路径 B：每个 Block 一个 K8s Job

#### 2.4.1 筛选可执行 Block：__schedule_blocks()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L601-L670

```python
def __schedule_blocks(self, block_runs=None):
    self.pipeline_run.update_block_run_statuses(self.pipeline_run.initial_block_runs)
    block_runs_to_schedule = self.pipeline_run.executable_block_runs(...)

    for b in block_runs_to_schedule[:block_run_quota]:
        b.update(status=BlockRun.BlockRunStatus.QUEUED)   # INITIAL → QUEUED
        job_manager.add_job(           # Mage 内部进程级 Job 管理器
            JobType.BLOCK_RUN, b.id,
            run_block,                  # 进程池回调
            self.pipeline_run.id, b.id,
            self.pipeline_run.get_variables(),
            self.build_tags(block_run_id=b.id, block_uuid=b.block_uuid),
        )
```

#### 2.4.2 Block 进程池执行函数：run_block()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1267-L1333

```python
def run_block(pipeline_run_id, block_run_id, variables, tags, ...):
    block_run = BlockRun.query.get(block_run_id)
    block_run.update(status=BlockRun.BlockRunStatus.RUNNING, started_at=...)  # QUEUED → RUNNING

    pipeline_scheduler = PipelineScheduler(pipeline_run)

    # ★ 准备回调函数，这些回调将在 K8s Job 完成后于当前 Worker 进程中执行
    if schedule_after_complete:
        on_complete = pipeline_scheduler.on_block_complete           # 会触发重新调度
    else:
        on_complete = pipeline_scheduler.on_block_complete_without_schedule
    on_failure = pipeline_scheduler.on_block_failure

    return ExecutorFactory.get_block_executor(
        pipeline, block_uuid, block_run_id=block_run.id,
        execution_partition=execution_partition,
    ).execute(
        block_run_id=block_run.id, global_vars=variables,
        on_complete=on_complete,     # ★ 传入回调
        on_failure=on_failure,       # ★ 传入回调
        pipeline_run_id=pipeline_run_id,
    )
```

#### 2.4.3 K8sBlockExecutor._execute()

文件：`mage_ai/data_preparation/executors/k8s_block_executor.py`，L30-L60

与 `K8sPipelineExecutor.execute()` 逻辑一致，区别在于：
- Job 命名：`{cluster_uuid}-{prefix}-block-{block_run_id}`
- CLI 命令多了 `--block-uuid` 和 `--block-run-id`

### 2.5 K8s Job 对象构建与提交

文件：`mage_ai/services/k8s/job_manager.py`，L67-L230

#### 2.5.1 配置加载

1. 优先 `load_incluster_config()`（Pod 内运行），回退到 `load_kube_config()`
2. 读取当前 Mage Server Pod 的规格（`read_namespaced_pod`），作为 Job Pod 的继承模板

#### 2.5.2 Pod/Container 规格合并

- `merge_pod_spec()`（L155-L179）：用户 pod_config + Mage Server Pod 的 volumes / tolerations / node_selector / scheduler_name / image_pull_secrets
- `merge_container_spec()`（L181-L204）：用户 container_config + Mage Server Container 的 env / env_from / volume_mounts / image

合并策略：用户非 None 优先；列表类字段取并集去重。

#### 2.5.3 Job 配置

- `backoff_limit`：默认 `0`，K8s Job 失败不重试
- `active_deadline_seconds` / `ttl_seconds_after_finished`：可通过 `job_config` 配置

#### 2.5.4 提交到 K8s API Server

`create_job()`（L224-L229）调用 `batch_api_client.create_namespaced_job()` 完成提交。

**【入口协作阶段结束】**：此时 K8s Job 已成功创建并提交，等待 K8s 调度 Pod。

---

## 三、状态推进：从 Pod 运行到 DB 状态落库与调度聚合

> **边界定义**：状态推进阶段从 K8s Pod 开始运行（进入 Running 状态）起，到 BlockRun/PipelineRun 状态全部更新到 DB、调度器完成下一轮 Block 的调度聚合为止。
>
> 核心特征是**双端状态写入**：Pod 内部尝试写一次（DB 直写），Pod 外部回调再写一次（幂等）并触发调度推进。但双端写入都有各自的失败条件和兜底限制，详见 3.6。

### 3.1 状态枚举

文件：`mage_ai/orchestration/db/models/schedules.py`

**PipelineRunStatus**（L769-L774）：
```
INITIAL → RUNNING → COMPLETED
                    → FAILED
                    → CANCELLED
```

**BlockRunStatus**（L1664-L1672）：
```
INITIAL → QUEUED → RUNNING → COMPLETED
                               → FAILED
                               → CANCELLED
                               → UPSTREAM_FAILED
                               → CONDITION_FAILED
```

### 3.2 K8s Job 轮询（Pod 外部，Mage Worker 进程侧）

文件：`mage_ai/services/k8s/job_manager.py`，L67-L98

```python
def run_job(self, command, k8s_config=None):
    if not self.job_exists():
        job = self.create_job_object(command, k8s_config=k8s_config)
        self.create_job(job)       # 提交到 K8s API Server

    job_completed = False
    while not job_completed:
        # 每 5 秒轮询一次 K8s Job 状态（同步阻塞）
        api_response = self.batch_api_client.read_namespaced_job(
            name=self.job_name, namespace=self.namespace
        )
        if api_response.status.succeeded is not None or \
                api_response.status.failed is not None:
            job_completed = True
        time.sleep(5)

    self.delete_job()              # ★ 正常完成路径下主动删除 K8s Job
    if api_response.status.succeeded is None:
        raise Exception(f'Failed to execute k8s job {self.job_name}')
```

`run_job()` 是**同步阻塞**的：调用方线程（Mage Worker 进程）会一直卡在 while 循环中，直到 K8s Job 完成（成功或失败）。

### 3.3 Pod 内部执行与状态回写（Pod 内部，CLI 侧）

K8s Job 创建的 Pod 启动后执行：

```bash
/app/run_app.sh mage run <repo_path> <pipeline_uuid> \
  --executor-type local_python \
  [--block-uuid <block_uuid>] \
  [--block-run-id <block_run_id>] \
  [--pipeline-run-id <pipeline_run_id>]
```

#### 3.3.1 CLI 入口

文件：`mage_ai/cli/main.py`，L160-L276

```python
def run(project_path, pipeline_uuid, block_uuid=None, block_run_id=None,
        pipeline_run_id=None, executor_type=None, callback_url=None, ...):
    pipeline = Pipeline.get(pipeline_uuid, repo_path=project_path)

    if block_uuid is None:
        # ========== 整 Pipeline 执行 ==========
        ExecutorFactory.get_pipeline_executor(
            pipeline, execution_partition=execution_partition,
            executor_type=executor_type,       # Pod 内传入的是 local_python
        ).execute(
            global_vars=global_vars, pipeline_run_id=pipeline_run_id,
            update_status=False,
            # ★ 没有 on_complete / on_failure 回调（Pod 内没有调度器上下文）
        )
    else:
        # ========== 单个 Block 执行 ==========
        ExecutorFactory.get_block_executor(
            pipeline, block_uuid, block_run_id=block_run_id,
            execution_partition=execution_partition,
            executor_type=executor_type,       # Pod 内传入的是 local_python
        ).execute(
            block_run_id=block_run_id,
            global_vars=global_vars,
            pipeline_run_id=pipeline_run_id,
            callback_url=callback_url,          # ★ 传了 callback_url，但见 3.6
            update_status=False,
            # ★ 没有 on_complete / on_failure 回调
        )
```

> **边界要点 1**：CLI 入口**不传递** `on_complete` / `on_failure` 回调。Pod 内的执行器没有调度器回调可用，只能通过 DB 直写（或 callback_url 兜底）更新状态。

#### 3.3.2 Pod 内 Block 状态写入：__update_block_run_status()

文件：`mage_ai/data_preparation/executors/block_executor.py`，L1369-L1438

```python
def __update_block_run_status(
    self, status, block_run_id=None, callback_url=None,
    error_details=None, pipeline_run=None, tags=None,
):
    if not block_run_id and not callback_url:
        return                    # ★ 两者都为空则直接返回，状态丢失

    try:
        if not block_run_id:
            block_run_id = int(callback_url.split('/')[-1])
        block_run = BlockRun.query.get(block_run_id)
        update_kwargs = dict(status=status)
        if status == BlockRun.BlockRunStatus.COMPLETED:
            update_kwargs['completed_at'] = datetime.now(tz=pytz.UTC)
        block_run.update(**update_kwargs)    # ★ 主路径：直接写 DB
        return
    except Exception as err2:
        self.logger.exception(f'Failed to update block run status to {status}...', error=err2)

    # ★★ 兜底路径：DB 写入失败，回退到 HTTP PUT callback_url ★★
    block_run_data = dict(status=status)
    if error_details:
        block_run_data['error_details'] = error_details
    response = requests.put(
        callback_url,
        data=json.dumps({'block_run': block_run_data}),
        headers={'Content-Type': 'application/json'},
    )
```

状态写入触发点（同文件 BlockExecutor.execute()）：
- **RUNNING**（L121-L126）：由 Pod 外 `run_block()` 在提交 K8s Job 前已写入；Pod 内部不重复写
- **COMPLETED**（L739-L745）：`_execute()` 成功返回后，若 `on_complete` 为空则调用
- **FAILED**（L687-L695）：异常捕获后，若 `on_failure` 为空则调用
- **CONDITION_FAILED**（L460-L465）：条件判断不满足

> **边界要点 2**：Pod 内执行时 `on_complete` 和 `on_failure` 均为 `None`，所以状态一定走 `__update_block_run_status()` → DB 直写路径。

#### 3.3.3 Pod 内整 Pipeline 执行时的 Block 调度

如果是整 Pipeline 的 K8s Job，Pod 内会启动 `PipelineExecutor.execute()`（基类），内部异步执行所有 Block（`mage_ai/data_preparation/executors/pipeline_executor.py` L94-L172）。每个 Block 的状态写入同样走 DB 直写。

### 3.4 Pod 外部回调与调度推进（Pod 外部，Mage Worker 进程侧）

K8s Job 完成后（succeeded=1 或 failed≠None），`JobManager.run_job()` 返回（或抛异常），控制权回到 Pod 外部的 `BlockExecutor.execute()` 框架。

#### 3.4.1 成功路径回调

文件：`mage_ai/data_preparation/executors/block_executor.py`，L728-L745

```python
if should_finish:
    if on_complete is not None:
        # ★ Pod 外部：调用传入的回调（run_block 中绑定的 PipelineScheduler.on_block_complete）
        on_complete(self.block_uuid)
    else:
        # Pod 内部：DB 直写（Pod 内 on_complete 为 None，走此分支）
        self.__update_block_run_status(BlockRun.BlockRunStatus.COMPLETED, ...)
```

`on_complete` 回调对应 `PipelineScheduler.on_block_complete()`：

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L377-L410

```python
def on_block_complete(self, block_uuid, metrics=None):
    block_run = BlockRun.get(pipeline_run_id=self.pipeline_run.id, block_uuid=block_uuid)

    @retry(retries=2, delay=5)    # ★ 共 3 次 DB 写入机会（1 次正跑 + 2 次重试）
    def update_status(metrics=metrics):
        block_run.update(
            status=BlockRun.BlockRunStatus.COMPLETED,
            completed_at=datetime.now(tz=pytz.UTC),
            metrics=...
        )
    update_status()

    self.pipeline_run.refresh()
    if self.pipeline_run.status != PipelineRun.PipelineRunStatus.RUNNING:
        return
    # ★★★ 核心职责：触发下一轮调度，选出下一批可执行 Block ★★★
    self.schedule()
```

> **边界要点 3**：Pod 外部回调的核心职责**不是**写状态（那是 Pod 内优先尝试的），而是**调用 `self.schedule()` 推进整个 Pipeline 的调度**。状态写入是附带的幂等操作，保证即使 Pod 内写入失败也能在调度侧补写。

#### 3.4.2 失败路径回调

文件：`mage_ai/data_preparation/executors/block_executor.py`，L648-L706

```python
except Exception as error:
    if on_failure is not None:
        # ★ Pod 外部：调用传入的回调
        on_failure(self.block_uuid, error=error_details)
    else:
        # Pod 内部：DB 直写
        self.__update_block_run_status(BlockRun.BlockRunStatus.FAILED, ...)
    raise error
```

`on_failure` 回调对应 `PipelineScheduler.on_block_failure()`：

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L443-L488

```python
def on_block_failure(self, block_uuid, **kwargs):
    block_run = BlockRun.get(pipeline_run_id=self.pipeline_run.id, block_uuid=block_uuid)
    metrics = block_run.metrics or {}
    if error:
        metrics['error'] = dict(error=..., errors=..., message=...)  # 异常类型、堆栈、traceback
    block_run.update(metrics=metrics, status=BlockRun.BlockRunStatus.FAILED)
    # ★ 注意：on_block_failure 没有 @retry 装饰器，只有 1 次写入机会

    if not self.allow_blocks_to_fail and PipelineType.INTEGRATION == self.pipeline.type:
        job_manager.kill_pipeline_run_job(self.pipeline_run.id)
        for stream in self.streams:
            job_manager.kill_integration_stream_job(...)
```

### 3.5 PipelineRun 状态聚合

PipelineRun 的最终状态由调度器在下一轮 `schedule()` 中聚合：

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L240-L331

```python
def schedule(self, ...):
    # ...
    if self.pipeline_run.all_blocks_completed(self.allow_blocks_to_fail):
        if self.pipeline_run.any_blocks_failed():
            self.pipeline_run.update(                    # 有 Block 失败 → FAILED
                status=PipelineRun.PipelineRunStatus.FAILED,
                completed_at=datetime.now(tz=pytz.UTC),
            )
            self.on_pipeline_run_failure(error_msg)
        else:
            self.pipeline_run.complete()                  # 全部成功 → COMPLETED
            self.notification_sender.send_pipeline_run_success_message(...)
            UsageStatisticLogger().pipeline_run_ended_sync(self.pipeline_run)
```

> **边界要点 4**：整 Pipeline K8s Job 完成后，Pod 外部的 Worker 进程从 `K8sPipelineExecutor.execute()` 阻塞中返回，随后由外部调度循环（周期性调用 `PipelineScheduler.schedule()`）检测所有 BlockRun 状态并完成 PipelineRun 聚合。Block 级 K8s Job 的聚合则由 `on_block_complete` → `self.schedule()` 直接触发。

### 3.6 双端状态写入的失败条件与兜底限制

双端写入并非完全可靠，每一端都有明确的失败条件和兜底上限：

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     Pod 内部状态写入（__update_block_run_status）                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  主路径：DB 直写 block_run.update(**kwargs)                                           │
│    ├─ 成功：直接 return                                                              │
│    └─ 失败（DB 连接异常、SQL 错误等） → 进入 except                                    │
│                                                                                      │
│  兜底路径：HTTP PUT callback_url                                                      │
│    ├─ 前置条件：callback_url 必须非 None                                              │
│    │                                                                                  │
│    │  ★ 关键限制 ★                                                                   │
│    │  BlockExecutor._run_commands() (mage_ai/data_preparation/executors/             │
│    │  block_executor.py L1324-L1367) 构建 Pod 内 CLI 命令时，**不拼接                │
│    │  --callback-url 参数**；K8sBlockExecutor._execute() 也没额外传入。                │
│    │  因此 Pod 内执行时 callback_url 几乎总是 None（除非用户手动传）。                  │
│    │                                                                                  │
│    ├─ callback_url is None → 函数开头 L1390-1391 `if not block_run_id and            │
│    │   not callback_url: return` 直接返回，**状态写入完全丢失**                        │
│    └─ callback_url 非 None → 发送 HTTP PUT，但可能因网络/API 问题失败，代码无再重试    │
│                                                                                      │
│  极端后果：Pod 内业务执行成功，但 DB 连不上 + callback_url=None → BlockRun 永远        │
│           停在 RUNNING 状态，后续只能靠 Pod 外回调或外部人工修复                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                   Pod 外部回调写入（on_block_complete / on_block_failure）            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  on_block_complete：                                                                 │
│    ├─ @retry(retries=2, delay=5) → 共 3 次 DB 写入机会                                │
│    ├─ 3 次全失败：抛出异常，后续 self.schedule() 不会执行，Pipeline 调度卡住           │
│    └─ 成功写入后：self.schedule() 推进下一批 Block 的调度                              │
│                                                                                      │
│  on_block_failure：                                                                  │
│    ├─ 无 @retry 装饰器 → 仅 1 次 DB 写入机会                                          │
│    ├─ 写入失败：异常向上传播，BlockRun 可能停在 RUNNING                                │
│    └─ 集成管道模式：kill 其他关联 Job                                                  │
│                                                                                      │
│  极端后果：Worker 进程在回调执行前被 SIGKILL（见四），则回调完全不执行，Pod 外写入丢失  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

> **边界要点 5**：双端写入是「尽量写」而非「强一致保证」。Pod 内优先写，Pod 外回调做幂等补写并推进调度；但两端各有其失败模式，状态最终一致性依赖至少一端写入成功。

**【状态推进阶段结束】**：所有 BlockRun 状态已尽量落库，PipelineRun 状态已聚合，后续 Block 的调度决策已完成。

---

## 四、取消收尾：从触发取消到资源清理

> **边界定义**：取消收尾阶段从外部触发取消（或超时、或 K8s Job 失败）开始，到 Mage 内部进程级 Job 被 kill、DB 中 BlockRun/PipelineRun 状态更新为最终状态为止。
>
> **重要边界**：按 Block 调度的 K8s Job **不会被立即删除**；K8s 层面的 Job 和 Pod 清理不处于此阶段的必然同步路径上。

### 4.1 取消入口（共 4 个）

#### 入口 1：用户主动取消 PipelineRun

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1426-L1457

```python
def stop_pipeline_run(pipeline_run, pipeline=None,
                       status=PipelineRun.PipelineRunStatus.CANCELLED):
    if pipeline_run.status not in [INITIAL, RUNNING]:   # 幂等校验
        return
    pipeline_run.update(status=status)
    UsageStatisticLogger().pipeline_run_ended_sync(pipeline_run)
    cancel_block_runs_and_jobs(pipeline_run, pipeline)
```

#### 入口 2：Pipeline 超时

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L528-L558

```python
def __check_pipeline_run_timeout(self) -> bool:
    if time_difference > int(pipeline_run_timeout):
        status = self.pipeline_schedule.timeout_status or FAILED
        self.pipeline_run.update(status=status)
        self.on_pipeline_run_failure('Pipeline run timed out.', status=status)
```

#### 入口 3：Block 超时

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L567-L599

```python
def __check_block_run_timeout(self) -> bool:
    if time_difference > int(block.timeout):
        block_executor.logger.error('Block timed out...')
        self.on_block_failure(block_run.block_uuid)
        job_manager.kill_block_run_job(block_run.id)   # kill Mage 内部进程级 Job
```

#### 入口 4：K8s Job 执行失败

见 3.4.2，`JobManager.run_job()` 抛异常 → `on_failure` 回调。

### 4.2 核心取消逻辑：cancel_block_runs_and_jobs()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1460-L1519

```python
def cancel_block_runs_and_jobs(pipeline_run, pipeline=None):
    # ========== 步骤 1：批量更新 BlockRun 状态 ==========
    block_runs_to_cancel = [
        b for b in pipeline_run.block_runs
        if b.status in [INITIAL, QUEUED, RUNNING]
    ]
    BlockRun.batch_update_status(
        [b.id for b in block_runs_to_cancel],
        BlockRun.BlockRunStatus.CANCELLED
    )

    running_blocks = [b for b in block_runs_to_cancel if b.status == RUNNING]

    # ========== 步骤 2：Kill Mage 内部进程级 Job ==========
    job_manager = get_job_manager()   # Mage 内部 JobManager（非 K8s）

    if pipeline and (pipeline.type in [INTEGRATION, STREAMING]
                     or pipeline.run_pipeline_in_one_process):
        # ── 整 Pipeline / 流式 / 集成管道 ──
        job_manager.kill_pipeline_run_job(pipeline_run.id)
        if pipeline.type == INTEGRATION:
            for stream in pipeline.streams():
                job_manager.kill_integration_stream_job(...)

        # ★ 整 Pipeline K8s 执行：显式调用 cancel() 删除 K8s Job ★
        if pipeline_run.executor_type == ExecutorType.K8S:
            ExecutorFactory.get_pipeline_executor(
                pipeline, executor_type=pipeline_run.executor_type,
            ).cancel(pipeline_run_id=pipeline_run.id)

    else:
        # ── 按 Block 调度（默认批处理） ──
        for b in running_blocks:
            job_manager.kill_block_run_job(b.id)
        # ★★ 注意：此处 **没有** 对应的 K8sJobManager.delete_job() 调用 ★★
        # K8s 层面的 Job 和 Pod 不会被此函数同步删除

    # ========== 步骤 3：异步清理 GenericJob ==========
    GenericJob.enqueue_cancel_pipeline_run(pipeline_run.id, cancelled_block_run_ids)
```

### 4.3 Mage 内部 kill_job 的实际行为

文件：`mage_ai/orchestration/queue/process_queue.py`，L161-L183

```python
def kill_job(self, job_id: str):
    print(f'Kill job {job_id}, job_dict {self.job_dict}')
    job = self.job_dict.get(job_id)
    if not job:
        self.__set_kill_job(job_id)
        return
    if isinstance(job, int):          # job == PID → 正在运行中
        if job == os.getpid():
            self.job_dict[job_id] = JobStatus.CANCELLED
        try:
            os.kill(job, signal.SIGKILL)    # ★ 发送 SIGKILL，强制终止进程
        except Exception as err:
            print(err)
    self.job_dict[job_id] = JobStatus.CANCELLED
    self.__unset_kill_job(job_id)
```

> **边界要点 6**：`kill_block_run_job` 最终调用 `os.kill(pid, signal.SIGKILL)`。SIGKILL 不能被捕获、阻塞或忽略，目标进程（Mage Worker）立即终止，**没有任何机会执行清理逻辑**（包括 `K8sJobManager.run_job()` 末尾的 `delete_job()`）。

### 4.4 按 Block 调度的 K8s Job/Pod 实际删除时机

```
按 Block 调度取消触发
        │
        ▼
cancel_block_runs_and_jobs()
        │
        ├─► BlockRun 状态 → CANCELLED（DB 已更新）
        │
        └─► job_manager.kill_block_run_job(block_run_id)
                   │
                   ▼
           ProcessQueue.kill_job()
                   │
                   ▼
           os.kill(worker_pid, SIGKILL)   ← Mage Worker 被强杀
                   │
                   │  Worker 正阻塞在 K8sJobManager.run_job() 的 while 循环中
                   │  ┌─────────────────────────────────────────────┐
                   │  │  while not job_completed:                    │
                   │  │      read_namespaced_job()   ← 被 SIGKILL 中断
                   │  │      time.sleep(5)
                   │  │  self.delete_job()              ← 永远不会执行
                   │  └─────────────────────────────────────────────┘
                   │
                   ▼
         ┌───────────────────────────────────────────┐
         │  K8s Job 和 Pod 的实际命运                  │
         │                                             │
         │  1. 如果配置了 ttl_seconds_after_finished    │
         │     → 等待 TTL 到期后 K8s 自动清理 Job       │
         │     （Pod 随之被级联删除）                    │
         │                                             │
         │  2. 如果配置了 active_deadline_seconds       │
         │     → Job 超时被 K8s 标记为 Failed           │
         │     → 但 Job/Pod 仍不自动删除（除非有 TTL）   │
         │                                             │
         │  3. Pod 内部业务逻辑自然跑完                  │
         │     → Job succeeded=1 但 delete_job 未执行   │
         │     → Job/Pod 残留，直到 TTL 或手动清理       │
         │                                             │
         │  4. 无 TTL、业务不跑完、无 active_deadline    │
         │     → Job 和 Pod 永久残留，需人工 kubectl delete │
         └───────────────────────────────────────────┘
```

> **边界要点 7**：按 Block 调度的 K8s 取消是「**异步尽力清理**」模型。同步路径只保证 DB 状态正确和 Mage Worker 进程终止；K8s Job/Pod 的删除依赖 TTL、超时配置或外部人工干预，**不在取消的同步必然路径上**。

### 4.5 整 Pipeline K8s 执行的取消（有显式清理）

文件：`mage_ai/data_preparation/executors/k8s_pipeline_executor.py`，L30-L42

```python
def cancel(self, pipeline_run_id=None):
    if pipeline_run_id is None:
        return
    try:
        job_manager = self.get_job_manager(pipeline_run_id=pipeline_run_id)
        job_manager.delete_job()
    except Exception:
        traceback.print_exc()       # 容忍 Job 已不存在
```

文件：`mage_ai/services/k8s/job_manager.py`，L231-L241

```python
def delete_job(self):
    self.batch_api_client.delete_namespaced_job(
        name=self.job_name,
        namespace=self.namespace,
        body=client.V1DeleteOptions(
            propagation_policy='Foreground',   # 级联删除：先删 Pod，再删 Job
            grace_period_seconds=0,            # 零宽限期，立即终止
        )
    )
```

`Foreground` 级联策略保证 Pod 在 Job 之前被删除；`grace_period_seconds=0` 跳过优雅终止，立即发送 SIGKILL。

### 4.6 失败通知与资源上报

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L341-L374

```python
def on_pipeline_run_failure(self, error_msg, status=FAILED):
    UsageStatisticLogger().pipeline_run_ended_sync(self.pipeline_run)
    if status == FAILED:
        stacktrace = f'Error for block {br.block_uuid}:\n{message}'
        self.notification_sender.send_pipeline_run_failure_message(
            pipeline=self.pipeline, pipeline_run=self.pipeline_run,
            error=error_msg, stacktrace=stacktrace,
        )
    cancel_block_runs_and_jobs(self.pipeline_run, self.pipeline)
```

### 4.7 Job 自动清理的两道防线（仅正常完成路径）

1. **主动删除**：`JobManager.run_job()` 在轮询检测到 Job 结束后，无论成功失败都会立即调用 `delete_job()`（`mage_ai/services/k8s/job_manager.py` L95）
2. **被动过期**：K8s Job 配置了 `ttl_seconds_after_finished`（可通过 `job_config` 设置），K8s Controller 会在 Job 完成后指定秒数自动清理

> 这两道防线仅在 Worker 进程正常跑完 `run_job()` 全流程时生效。如果 Worker 被 SIGKILL 中断，两道防线均不触发。

**【取消收尾阶段结束】**：DB 中 PipelineRun/BlockRun 已处于最终状态（COMPLETED / FAILED / CANCELLED），Mage Worker 进程已被终止；但 K8s Job/Pod 可能仍存在，取决于 TTL 等配置。

---

## 五、完整时序图（Block 级 K8s 执行 + 取消场景）

```
用户取消请求          PipelineScheduler   ProcessQueue    Mage Worker      K8sJobManager    K8s API      Pod (CLI)
     │                     │                │              │                  │              │             │
     │ stop_pipeline_run() │                │              │                  │              │             │
     │────────────────────>│                │              │                  │              │             │
     │                     │ BlockRun → CANCELLED (批量)    │                  │              │             │
     │                     │                │              │                  │              │             │
     │                     │ kill_block_run_job(block_id)   │                  │              │             │
     │                     │───────────────>│              │                  │              │             │
     │                     │                │ os.kill(SIGKILL)                │              │             │
     │                     │                │─────────────>│ (进程被强杀)      │              │             │
     │                     │                │              │                  │              │             │
     │                     │                │              │  ┌── Worker 正在执行的代码 ──┐    │             │
     │                     │                │              │  │  K8sJobManager.run_job():   │    │             │
     │                     │                │              │  │    while not completed:     │    │             │
     │                     │                │              │  │        read_job()  ←──被 SIGKILL 中断        │
     │                     │                │              │  │        sleep(5)             │    │             │
     │                     │                │              │  │    delete_job()  ←── ★ 永远不执行          │
     │                     │                │              │  └────────────────────────────┘    │             │
     │                     │                │              │                  │              │             │
     │                     │                │              │                  │  ┌── K8s Job/Pod 命运 ──┐        │
     │                     │                │              │                  │  │ 1. ttl 到期自动清理    │        │
     │                     │                │              │                  │  │ 2. active_deadline 超时│        │
     │                     │                │              │                  │  │ 3. 业务跑完后残留      │        │
     │                     │                │              │                  │  │ 4. 永久残留直到手动删  │        │
     │                     │                │              │                  │  └───────────────────────┘        │
```

---

## 六、关键设计要点总结

### 6.1 两层执行器 + 双端写入模型

```
┌──────────────────────────────────────────────────────────┐
│  调度层 Executor（Pod 外，Mage Worker 进程）               │
│  K8sPipelineExecutor / K8sBlockExecutor                   │
│    职责：提交 K8s Job、轮询状态、触发 on_complete 回调      │
│          → 回调负责推进调度（self.schedule()）             │
│          → on_block_complete 有 3 次 DB 写入重试          │
│          → on_block_failure 仅 1 次 DB 写入机会           │
└──────────────────────┬───────────────────────────────────┘
                       │ K8s API
                       ▼
┌──────────────────────────────────────────────────────────┐
│  执行层 Executor（Pod 内，CLI 启动）                       │
│  PipelineExecutor / BlockExecutor（type=local_python）     │
│    职责：执行业务逻辑、DB 直写 BlockRun 状态                │
│          → 主路径：block_run.update()                     │
│          → 兜底：callback_url HTTP PUT（但 K8s 场景下      │
│            _run_commands() 不传 callback_url，基本为 None）│
│          → 两端都失败则状态写入丢失                        │
└──────────────────────────────────────────────────────────┘
```

### 6.2 三个阶段的清晰边界

| 阶段 | 起点 | 终点 | 必然完成的操作 | 可能异步/缺失的操作 |
|------|------|------|--------------|-------------------|
| **入口协作** | PipelineRun.status = RUNNING | K8s Job 成功提交到 API Server | DB 状态更新、Mage Job 入队、K8s Job 创建 | — |
| **状态推进** | Pod 开始执行业务逻辑 | 调度器完成 PipelineRun 聚合 + 下一轮 Block 调度决策 | Pod 内尝试 DB 直写、Pod 外回调尝试 DB 补写并调度 | 双端写入可能各自失败，依赖至少一端成功 |
| **取消收尾** | 取消/超时/失败信号产生 | DB 状态最终化 + Mage Worker 进程终止 | BlockRun/PipelineRun 状态更新、Mage Worker SIGKILL | 按 Block 调度的 K8s Job/Pod **不保证同步删除**，依赖 TTL/超时/手动 |

### 6.3 按 Block 调度取消的核心边界

- `cancel_block_runs_and_jobs()` 对按 Block 调度模式**仅 kill Mage 内部进程级 Job**，不调用 K8s API
- Worker 被 SIGKILL 强杀后，`K8sJobManager.run_job()` 末尾的 `delete_job()` 永远不执行
- K8s Job/Pod 的删除是**异步非必然**的，取决于 `ttl_seconds_after_finished`、`active_deadline_seconds` 或外部干预
- 对比：整 Pipeline K8s 执行模式有显式 `K8sPipelineExecutor.cancel()` → `delete_job()`，可同步清理

### 6.4 双端状态写入的兜底限制

- Pod 内兜底路径 `callback_url` 在 K8s 场景下几乎总是 None（`_run_commands()` 不拼接该参数），Pod 内实际上只有 DB 直写一次机会
- Pod 外 `on_block_complete` 有 `@retry(retries=2, delay=5)` 共 3 次机会，`on_block_failure` 无重试
- 如果 Worker 被 SIGKILL（取消场景），Pod 外回调完全不执行，只能依赖 Pod 内写入
- 极端情况下（Pod 内 DB 不可达 + Pod 外 Worker 被强杀），BlockRun 会永久停在 RUNNING，需人工修复

### 6.5 同步阻塞与并发解耦

`JobManager.run_job()` 的 while 轮询是同步阻塞的，调用方线程被占用直到 K8s Job 完成。这简化了编程模型，但并发能力依赖 Mage 内部 `ProcessQueue` 的 Worker 池大小（默认 CPU 核数）。

### 6.6 取消的幂等保护

所有取消入口（`stop_pipeline_run`、超时检测、失败回调）都会先检查 `pipeline_run.status in [INITIAL, RUNNING]`，避免重复取消。`K8sPipelineExecutor.cancel()` 和 `JobManager.delete_job()` 也都有 try-except 保护，容忍 Job 已不存在的情况。

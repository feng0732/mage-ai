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
| K8s 常量 | `mage_ai/services/k8s/constants.py` | 命名空间、环境变量名等常量 |
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
    # 步骤 1：为 PipelineRun 创建对应的 BlockRun 记录
    self.pipeline_run.create_block_runs()
    # 步骤 2：PipelineRun 状态 INITIAL → RUNNING
    self.pipeline_run.update(
        started_at=datetime.now(tz=pytz.UTC),
        status=PipelineRun.PipelineRunStatus.RUNNING,
    )
    # 步骤 3：进入调度分支
    if should_schedule:
        self.schedule()
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

#### 2.3.1 提交进程级 Job：__schedule_pipeline()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L836-L869

```python
def __schedule_pipeline(self):
    job_manager.add_job(
        JobType.PIPELINE_RUN,
        self.pipeline_run.id,
        run_pipeline,                  # 回调函数：在进程池线程中执行
        self.pipeline_run.id,          # 参数 1
        self.pipeline_run.get_variables(),  # 参数 2
        self.build_tags(),             # 参数 3
    )
```

> 注意：这里的 `job_manager` 是 Mage 内部的**进程/线程级 Job 管理器**，不是 K8s JobManager。它负责把 `run_pipeline` 函数丢到线程池中异步执行。

#### 2.3.2 执行函数：run_pipeline()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1336-L1362

```python
def run_pipeline(pipeline_run_id, variables, tags, allow_blocks_to_fail=False):
    pipeline_run = PipelineRun.query.get(pipeline_run_id)
    # 推断执行器类型并记录到 PipelineRun
    executor_type = ExecutorFactory.get_pipeline_executor_type(pipeline)
    pipeline_run.update(executor_type=executor_type)

    # 根据 executor_type 分派：K8S → K8sPipelineExecutor
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
    # 构建 K8s JobManager（注意：这才是 K8s 层面的 JobManager）
    job_manager = self.get_job_manager(
        pipeline_run_id=pipeline_run_id,
        global_vars=global_vars,
        **kwargs,
    )
    # 构建 Pod 内部执行的 CLI 命令
    cmd = self._run_commands(
        global_vars=global_vars,
        pipeline_run_id=pipeline_run_id,
        **kwargs
    )
    # 提交 K8s Job 并阻塞等待完成
    job_manager.run_job(cmd, k8s_config=self.executor_config)
```

Pod 内执行命令由基类 `mage_ai/data_preparation/executors/pipeline_executor.py` L181-L215 构建：

```bash
/app/run_app.sh mage run <repo_path> <pipeline_uuid> \
  --executor-type local_python \
  [--execution-partition <partition>] \
  [--pipeline-run-id <id>]
```

> **关键**：K8s Job 内部仍然使用 `local_python` 执行器。K8s 只负责把执行环境搬到独立 Pod，Pod 内部执行的就是常规的本地 Pipeline 流程。

#### 2.3.5 Job 命名

文件：`mage_ai/data_preparation/executors/k8s_pipeline_executor.py`，L65-L89

```
{ MAGE_CLUSTER_UUID } - { job_name_prefix } - pipeline - { pipeline_run_id }
```

例如：`cluster123-data-prep-pipeline-456`

`job_name_prefix` 支持 `{trigger_name}` 模板变量，会通过查询 PipelineRun → PipelineSchedule 关联的 Trigger 名称替换。

### 2.4 路径 B：每个 Block 一个 K8s Job

#### 2.4.1 筛选可执行 Block：__schedule_blocks()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L601-L670

```python
def __schedule_blocks(self, block_runs=None):
    # 先推进 INITIAL BlockRun 的状态（如依赖失败则标 UPSTREAM_FAILED）
    self.pipeline_run.update_block_run_statuses(self.pipeline_run.initial_block_runs)

    # 找出依赖全部满足、可以执行的 BlockRun
    block_runs_to_schedule = self.pipeline_run.executable_block_runs(
        allow_blocks_to_fail=self.allow_blocks_to_fail,
    )

    for b in block_runs_to_schedule[:block_run_quota]:
        # BlockRun: INITIAL → QUEUED
        b.update(status=BlockRun.BlockRunStatus.QUEUED)

        # 提交到 Mage 内部进程级 Job 管理器
        job_manager.add_job(
            JobType.BLOCK_RUN,
            b.id,
            run_block,               # 在进程池线程中执行
            self.pipeline_run.id,    # 参数
            b.id,
            self.pipeline_run.get_variables(),
            self.build_tags(block_run_id=b.id, block_uuid=b.block_uuid),
            ...
        )
```

#### 2.4.2 Block 执行函数：run_block()

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1267-L1333

```python
def run_block(pipeline_run_id, block_run_id, variables, tags, ...):
    pipeline_run = PipelineRun.query.get(pipeline_run_id)
    block_run = BlockRun.query.get(block_run_id)

    # BlockRun: QUEUED → RUNNING（在调度线程侧先标记）
    block_run.update(status=BlockRun.BlockRunStatus.RUNNING, started_at=...)

    pipeline_scheduler = PipelineScheduler(pipeline_run)

    # ★ 关键：准备回调函数，这些回调将在 K8s Job 完成后于调度线程侧执行
    if schedule_after_complete:
        on_complete = pipeline_scheduler.on_block_complete          # 会触发重新调度
    else:
        on_complete = pipeline_scheduler.on_block_complete_without_schedule
    on_failure = pipeline_scheduler.on_block_failure

    # 分派到 K8sBlockExecutor 并执行（阻塞）
    return ExecutorFactory.get_block_executor(
        pipeline, block_uuid,
        block_run_id=block_run.id,
        execution_partition=execution_partition,
    ).execute(
        block_run_id=block_run.id,
        global_vars=variables,
        on_complete=on_complete,      # ★ 传入回调
        on_failure=on_failure,        # ★ 传入回调
        pipeline_run_id=pipeline_run_id,
        ...
    )
```

#### 2.4.3 K8sBlockExecutor._execute()

文件：`mage_ai/data_preparation/executors/k8s_block_executor.py`，L30-L60

逻辑与 `K8sPipelineExecutor.execute()` 一致，区别仅在于：
- Job 命名：`{cluster_uuid}-{prefix}-block-{block_run_id}`
- CLI 命令多了 `--block-uuid` 和 `--block-run-id` 参数

### 2.5 K8s Job 对象构建与提交

文件：`mage_ai/services/k8s/job_manager.py`，L67-L230

#### 2.5.1 配置加载

JobManager 初始化时会：
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
> 核心特征是**双端状态写入**：Pod 内部写一次（DB 直写），Pod 外部回调再写一次（幂等）并触发调度推进。

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

### 3.2 K8s Job 轮询（Pod 外部，调度线程侧）

文件：`mage_ai/services/k8s/job_manager.py`，L67-L98

```python
def run_job(self, command, k8s_config=None):
    if not self.job_exists():
        job = self.create_job_object(command, k8s_config=k8s_config)
        self.create_job(job)          # 提交到 K8s

    job_completed = False
    while not job_completed:
        # 每 5 秒轮询一次 K8s Job 状态
        api_response = self.batch_api_client.read_namespaced_job(
            name=self.job_name, namespace=self.namespace
        )
        # succeeded 或 failed 非 None 表示 Job 已结束
        if api_response.status.succeeded is not None or \
                api_response.status.failed is not None:
            job_completed = True
        time.sleep(5)

    self.delete_job()                   # 主动清理 K8s Job
    if api_response.status.succeeded is None:
        raise Exception(f'Failed to execute k8s job {self.job_name}')
```

`run_job()` 是**同步阻塞**的：调用方线程（Mage 进程池线程）会一直卡在 while 循环中，直到 K8s Job 完成（成功或失败）。

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
        pipeline_run_id=None, executor_type=None, ...):
    pipeline = Pipeline.get(pipeline_uuid, repo_path=project_path)

    if block_uuid is None:
        # ========== 整 Pipeline 执行 ==========
        ExecutorFactory.get_pipeline_executor(
            pipeline,
            execution_partition=execution_partition,
            executor_type=executor_type,   # 注意：Pod 内传入的是 local_python
        ).execute(
            global_vars=global_vars,
            pipeline_run_id=pipeline_run_id,
            update_status=False,
            # ★ 注意：没有 on_complete / on_failure 回调
        )
    else:
        # ========== 单个 Block 执行 ==========
        ExecutorFactory.get_block_executor(
            pipeline, block_uuid,
            block_run_id=block_run_id,
            execution_partition=execution_partition,
            executor_type=executor_type,   # 注意：Pod 内传入的是 local_python
        ).execute(
            block_run_id=block_run_id,
            global_vars=global_vars,
            pipeline_run_id=pipeline_run_id,
            callback_url=callback_url,
            update_status=False,
            # ★ 注意：没有 on_complete / on_failure 回调！
        )
```

> **边界要点 1**：CLI 入口**不传递** `on_complete` / `on_failure` 回调。Pod 内的执行器没有调度器回调可用，只能通过 DB 直写更新状态。

#### 3.3.2 Pod 内 Block 状态写入：__update_block_run_status()

文件：`mage_ai/data_preparation/executors/block_executor.py`，L1369-L1428

```python
def __update_block_run_status(self, status, block_run_id=None, callback_url=None, ...):
    block_run = BlockRun.query.get(block_run_id)
    update_kwargs = dict(status=status)

    if status == BlockRun.BlockRunStatus.COMPLETED:
        update_kwargs['completed_at'] = datetime.now(tz=pytz.UTC)

    block_run.update(**update_kwargs)    # ★ 直接写 DB
    # 如果 DB 写入失败，回退到 callback_url HTTP PUT
```

状态写入触发点（同文件 BlockExecutor.execute()）：
- **RUNNING**（L121-L126）：执行 block 前，由 `run_block()` 在 Pod 外部已写入；Pod 内部不再重复写
- **COMPLETED**（L739-L745）：`_execute()` 成功返回后，若 `on_complete` 回调为空则调用
- **FAILED**（L687-L695）：异常捕获后，若 `on_failure` 回调为空则调用
- **CONDITION_FAILED**（L460-L465）：条件判断不满足

> **边界要点 2**：Pod 内执行时 `on_complete` 和 `on_failure` 均为 `None`，所以状态一定走 `__update_block_run_status()` → **DB 直写**路径。

#### 3.3.3 Pod 内整 Pipeline 执行时的 Block 调度

如果是整 Pipeline 的 K8s Job，Pod 内会启动 `PipelineExecutor.execute()`（基类），它内部异步执行所有 Block：

文件：`mage_ai/data_preparation/executors/pipeline_executor.py`，L94-L172

```python
async def __run_blocks(self, pipeline_run, allow_blocks_to_fail=False, global_vars=None):
    while not pipeline_run.all_blocks_completed(allow_blocks_to_fail):
        pipeline_run.update_block_run_statuses(pipeline_run.initial_block_runs)
        executable_block_runs = pipeline_run.executable_block_runs(...)

        block_run_tasks = [create_block_task(b, ...) for b in executable_block_runs]
        # create_block_task 内部调 BlockExecutor(block_run_id=...).execute(...)
        block_run_outputs = await asyncio.gather(*block_run_tasks)
```

此时每个 Block 内部的状态写入同样走 DB 直写。

### 3.4 Pod 外部回调与调度推进（Pod 外部，调度线程侧）

K8s Job 完成后（succeeded=1 或 failed≠None），`JobManager.run_job()` 返回（或抛异常），控制权回到 Pod 外部的 `BlockExecutor.execute()`（基类）框架。

#### 3.4.1 成功路径回调

文件：`mage_ai/data_preparation/executors/block_executor.py`，L728-L745

```python
if should_finish:
    if on_complete is not None:
        # ★ Pod 外部（调度线程）：调用传入的回调
        on_complete(self.block_uuid)
    else:
        # Pod 内部：DB 直写（Pod 内 on_complete 为 None，走此分支）
        self.__update_block_run_status(
            BlockRun.BlockRunStatus.COMPLETED, ...
        )
```

`on_complete` 回调是 `run_block()` 中绑定的 `PipelineScheduler.on_block_complete()`：

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L377-L410

```python
def on_block_complete(self, block_uuid, metrics=None):
    block_run = BlockRun.get(pipeline_run_id=self.pipeline_run.id, block_uuid=block_uuid)

    # ★ 再写一次状态（幂等），确保 completed_at 在调度侧有记录
    @retry(retries=2, delay=5)
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
    # ★★★ 关键：触发下一轮调度，选出下一批可执行 Block
    self.schedule()
```

> **边界要点 3**：Pod 外部回调的核心职责**不是**写状态（那是 Pod 内已完成的），而是**调用 `self.schedule()` 推进整个 Pipeline 的调度**，让后续依赖的 Block 得以进入调度队列。

#### 3.4.2 失败路径回调

文件：`mage_ai/data_preparation/executors/block_executor.py`，L648-L706

```python
except Exception as error:
    if on_failure is not None:
        # ★ Pod 外部（调度线程）：调用传入的回调
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
        # 把错误详情写入 metrics.error（包含异常类型、堆栈、traceback）
        metrics['error'] = dict(error=..., errors=..., message=...)
    block_run.update(metrics=metrics, status=BlockRun.BlockRunStatus.FAILED)

    if not self.allow_blocks_to_fail:
        # 集成管道：停止所有其他流
        if PipelineType.INTEGRATION == self.pipeline.type:
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
            # 有 Block 失败 → Pipeline FAILED
            self.pipeline_run.update(
                status=PipelineRun.PipelineRunStatus.FAILED,
                completed_at=datetime.now(tz=pytz.UTC),
            )
            self.on_pipeline_run_failure(error_msg)
        else:
            # 所有 Block 成功 → Pipeline COMPLETED
            self.pipeline_run.complete()
            self.notification_sender.send_pipeline_run_success_message(...)
            UsageStatisticLogger().pipeline_run_ended_sync(self.pipeline_run)
    # ...
```

> **边界要点 4**：整 Pipeline K8s Job 完成后，Pod 外部的调度线程从 `K8sPipelineExecutor.execute()` 的阻塞中返回，随后由外部调度循环（周期性调用 `PipelineScheduler.schedule()`）检测到所有 BlockRun 状态并完成 PipelineRun 聚合。Block 级 K8s Job 的聚合则由 `on_block_complete` → `self.schedule()` 直接触发。

### 3.6 双端状态写入总结

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        Block 级 K8s Job 状态推进                           │
├───────────────────────────────┬────────────────────────────────────────────┤
│        Pod 内部 (CLI)         │       Pod 外部 (调度线程)                  │
├───────────────────────────────┼────────────────────────────────────────────┤
│ mage run --executor-type      │ K8sBlockExecutor._execute() 阻塞返回       │
│   local_python                │                                            │
│                               │ BlockExecutor.execute() 框架判断 on_complete│
│ BlockExecutor.execute()       │   ↓                                        │
│   on_complete = None          │ on_complete(block_uuid)                    │
│   on_failure = None           │   ↓  PipelineScheduler.on_block_complete() │
│                               │   ① 幂等写 status=COMPLETED                 │
│ __update_block_run_status()   │   ② ★ self.schedule() 触发下一轮调度      │
│   → DB 直写 status            │                                            │
│   → COMPLETED / FAILED        │                                            │
│                               │ 失败路径：                                  │
│                               │ on_failure(block_uuid, error)              │
│                               │   ↓  PipelineScheduler.on_block_failure()  │
│                               │   ① 记录错误到 metrics.error                │
│                               │   ② status=FAILED                          │
│                               │   ③ 必要时 kill 其他 Job                   │
└───────────────────────────────┴────────────────────────────────────────────┘
```

**【状态推进阶段结束】**：所有 BlockRun 状态已落库，PipelineRun 状态已聚合，后续 Block 的调度决策已完成。

---

## 四、取消收尾：从触发取消到资源清理

> **边界定义**：取消收尾阶段从外部触发取消（或超时、或 K8s Job 失败）开始，到 K8s Job 被删除、Pod 被终止、DB 中所有 BlockRun/PipelineRun 状态更新为最终状态为止。

### 4.1 取消入口（共 4 个）

#### 入口 1：用户主动取消 PipelineRun

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L1426-L1457

```python
def stop_pipeline_run(pipeline_run, pipeline=None,
                       status=PipelineRun.PipelineRunStatus.CANCELLED):
    # 前置状态校验：只有 INITIAL / RUNNING 才允许取消
    if pipeline_run.status not in [INITIAL, RUNNING]:
        return
    # PipelineRun → CANCELLED（或自定义 status）
    pipeline_run.update(status=status)
    UsageStatisticLogger().pipeline_run_ended_sync(pipeline_run)
    # 取消所有 BlockRun 和 Job
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
        job_manager.kill_block_run_job(block_run.id)   # 杀 Mage 内部 Job
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

    # ========== 步骤 2：Kill Job（按执行模式区分） ==========
    job_manager = get_job_manager()   # Mage 内部进程级 JobManager

    if pipeline and (pipeline.type in [INTEGRATION, STREAMING]
                     or pipeline.run_pipeline_in_one_process):
        # ── 整 Pipeline / 流式 / 集成管道 ──
        job_manager.kill_pipeline_run_job(pipeline_run.id)  # 杀 Mage 内部 Job
        if pipeline.type == INTEGRATION:
            for stream in pipeline.streams():
                job_manager.kill_integration_stream_job(...)

        # ★★★ K8s 执行器取消：真正删除 K8s Job ★★★
        if pipeline_run.executor_type == ExecutorType.K8S:
            ExecutorFactory.get_pipeline_executor(
                pipeline,
                executor_type=pipeline_run.executor_type,
            ).cancel(pipeline_run_id=pipeline_run.id)

    else:
        # ── 按 Block 调度 ──
        for b in running_blocks:
            job_manager.kill_block_run_job(b.id)  # 逐个杀 Mage 内部 Job

    # ========== 步骤 3：异步清理 GenericJob ==========
    GenericJob.enqueue_cancel_pipeline_run(pipeline_run.id, cancelled_block_run_ids)
```

> **边界要点 5**：按 Block 调度的 K8s Job 取消没有对应分支（L1516-L1517 只 kill 了 Mage 内部进程级 Job），K8s 层面的 Block Job 需要依赖 `JobManager.run_job()` 内部的 `delete_job()` 或 K8s Job 的 `active_deadline_seconds` / `ttl_seconds_after_finished` 自动清理。

### 4.3 K8sPipelineExecutor.cancel()

文件：`mage_ai/data_preparation/executors/k8s_pipeline_executor.py`，L30-L42

```python
def cancel(self, pipeline_run_id=None):
    if pipeline_run_id is None:
        return
    try:
        job_manager = self.get_job_manager(pipeline_run_id=pipeline_run_id)
        job_manager.delete_job()
    except Exception:
        traceback.print_exc()   # 容忍 Job 已不存在的情况
```

### 4.4 JobManager.delete_job()

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

### 4.5 失败通知与资源上报

文件：`mage_ai/orchestration/pipeline_scheduler_original.py`，L341-L374

```python
def on_pipeline_run_failure(self, error_msg, status=FAILED):
    UsageStatisticLogger().pipeline_run_ended_sync(self.pipeline_run)
    if status == FAILED:
        # 从失败的 BlockRun.metrics.error 中取出 traceback
        stacktrace = f'Error for block {br.block_uuid}:\n{message}'
        self.notification_sender.send_pipeline_run_failure_message(
            pipeline=self.pipeline,
            pipeline_run=self.pipeline_run,
            error=error_msg,
            stacktrace=stacktrace,
        )
    # 取消尚未完成的 BlockRun 和 Job
    cancel_block_runs_and_jobs(self.pipeline_run, self.pipeline)
```

### 4.6 Job 自动清理的两道防线

1. **主动删除**：`JobManager.run_job()` 在轮询检测到 Job 结束后，无论成功失败都会立即调用 `delete_job()`（`mage_ai/services/k8s/job_manager.py` L95）
2. **被动过期**：K8s Job 配置了 `ttl_seconds_after_finished`（可通过 `job_config` 设置），K8s Controller 会在 Job 完成后指定秒数自动清理

两道防线确保 K8s 中不会残留大量已完成的 Job 对象。

**【取消收尾阶段结束】**：DB 中 PipelineRun/BlockRun 已处于最终状态（COMPLETED / FAILED / CANCELLED），K8s Job 和 Pod 已被删除，通知已发送。

---

## 五、完整时序图（Block 级 K8s 执行）

```
 外部调度循环       PipelineScheduler    Mage进程池     K8sBlockExecutor    K8s JobManager    K8s API      Pod (CLI)
      │                  │                │  (run_block)  │                    │               │             │
      │ schedule()       │                │               │                    │               │             │
      │─────────────────>│                │               │                    │               │             │
      │                  │ __schedule_blocks()             │                    │               │             │
      │                  │ block_run: INITIAL→QUEUED       │                    │               │             │
      │                  │ add_job(run_block)              │                    │               │             │
      │                  │───────────────>│                │                    │               │             │
      │                  │                │ block_run: QUEUED→RUNNING           │               │             │
      │                  │                │ get_block_executor(K8S)             │               │             │
      │                  │                │───────────────>│                    │               │             │
      │                  │                │                │ _execute()         │               │             │
      │                  │                │                │ get_job_manager()  │               │             │
      │                  │                │                │ _run_commands()    │               │             │
      │                  │                │                │ run_job(cmd)       │               │             │
      │                  │                │                │───────────────────>│               │             │
      │                  │                │                │                    │ create_job()  │             │
      │                  │                │                │                    │──────────────>│             │
      │                  │                │                │                    │   (Pod 创建)   │             │
      │                  │                │                │  while 轮询(5s)    │               │             │
      │                  │                │                │    read_job()      │──────────────>│             │
      │                  │                │                │                    │<──────────────│             │
      │                  │                │                │                    │               │ mage run    │
      │                  │                │                │                    │               │ --local_py  │
      │                  │                │                │                    │               │ BlockExec   │
      │                  │                │                │                    │               │  .execute() │
      │                  │                │                │                    │               │   │         │
      │                  │                │                │                    │               │   ├─ _execute() 业务逻辑
      │                  │                │                │                    │               │   │         │
      │                  │                │                │                    │               │   └─ __update_block_run_status()
      │                  │                │                │                    │               │      → DB 写 COMPLETED
      │                  │                │                │                    │               │             │
      │                  │                │                │                    │   succeeded=1  │             │
      │                  │                │                │                    │<──────────────│             │
      │                  │                │                │  delete_job()      │──────────────>│             │
      │                  │                │                │  (run_job 返回)    │               │             │
      │                  │                │                │<───────────────────│               │             │
      │                  │                │  BlockExec框架判断 on_complete      │               │             │
      │                  │                │─────────────────>│                    │               │             │
      │                  │                │  on_complete(block_uuid)            │               │             │
      │                  │<─────────────────────────────────────────────────────│               │             │
      │                  │ on_block_complete()                                 │               │             │
      │                  │   ① 幂等写 COMPLETED                                │               │             │
      │                  │   ② self.schedule()  ★ 调度下一批 Block             │               │             │
      │                  │─────────────────>│                    ... 继续循环 ...               │             │
```

---

## 六、关键设计要点总结

### 6.1 两层执行器 + 双端写入模型

```
┌──────────────────────────────────────────────────────┐
│  调度层 Executor（Pod 外，调度线程）                   │
│  K8sPipelineExecutor / K8sBlockExecutor               │
│    职责：提交 K8s Job、轮询状态、触发 on_complete 回调  │
│          → 回调负责推进调度（self.schedule()）         │
└──────────────────────┬───────────────────────────────┘
                       │ K8s API
                       ▼
┌──────────────────────────────────────────────────────┐
│  执行层 Executor（Pod 内，CLI 启动）                   │
│  PipelineExecutor / BlockExecutor（type=local_python）│
│    职责：执行业务逻辑、DB 直写 BlockRun 状态            │
│          → 无回调，只写数据库                          │
└──────────────────────────────────────────────────────┘
```

双端写入保证了即使某一端写入失败，状态最终仍能正确落库；同时通过 `on_complete` 回调把**调度推进**的职责严格留在调度线程侧。

### 6.2 三个阶段的清晰边界

| 阶段 | 起点 | 终点 | 主要发生位置 |
|------|------|------|-------------|
| 入口协作 | PipelineRun.status = RUNNING | K8s Job 成功提交到 API Server | Mage 调度线程 |
| 状态推进 | Pod 开始执行业务逻辑 | 调度器完成 PipelineRun 状态聚合及下一轮 Block 调度 | Pod 内部（DB 直写） + Mage 调度线程（回调 + 调度推进） |
| 取消收尾 | 取消/超时/失败信号产生 | DB 状态最终化 + K8s Job/Pod 全部删除 | Mage 调度线程 + K8s API |

### 6.3 同步阻塞与并发解耦

`JobManager.run_job()` 的 while 轮询是同步阻塞的，调用方线程被占用直到 K8s Job 完成。这简化了编程模型，但并发能力依赖 Mage 内部 `job_manager` 的线程池大小。K8s 本身的调度和 Pod 执行与调度线程完全解耦。

### 6.4 取消的幂等保护

所有取消入口（`stop_pipeline_run`、超时检测、失败回调）都会先检查 `pipeline_run.status in [INITIAL, RUNNING]`，避免重复取消。`K8sPipelineExecutor.cancel()` 和 `JobManager.delete_job()` 也都有 try-except 保护，容忍 Job 已不存在的情况。

### 6.5 配置继承保证环境一致

K8s Job Pod 的规格不是从零构建，而是从当前 Mage Server Pod 继承 volumes、env、image、tolerations、node_selector 等配置，再叠加用户自定义覆盖。这保证了 Job Pod 与 Mage Server 运行环境尽量一致，减少了「本地能跑 K8s 跑不了」类问题。

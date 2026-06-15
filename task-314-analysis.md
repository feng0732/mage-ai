# K8s 执行器代码链路深度分析

## 一、核心文件索引

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| Pipeline 级 K8s 执行器 | [k8s_pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/k8s_pipeline_executor.py) | 封装 Pipeline 在 K8s 上的执行与取消 |
| Block 级 K8s 执行器 | [k8s_block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/k8s_block_executor.py) | 封装单个 Block 在 K8s 上的执行 |
| 执行器工厂 | [executor_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/executor_factory.py) | 根据 executor_type 分派到对应执行器 |
| K8s Job 管理器 | [job_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py) | K8s Job 的创建、轮询、删除等底层操作 |
| K8s 配置 | [config.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/config.py) | K8sExecutorConfig 数据类，解析 Pod/Container/Job 配置 |
| K8s 常量 | [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/constants.py) | 命名空间、环境变量名等常量 |
| Pipeline 调度器 | [pipeline_scheduler_original.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py) | PipelineRun/BlockRun 的调度、状态推进、取消入口 |
| 状态模型 | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/db/models/schedules.py) | PipelineRun、BlockRun 数据模型及状态枚举 |
| CLI 入口 | [main.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/cli/main.py) | `mage run` 命令入口（K8s Pod 内部实际执行命令） |
| Pipeline 执行器基类 | [pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/pipeline_executor.py) | 提供 `_run_commands` 构建 CLI 命令 |
| Block 执行器基类 | [block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/block_executor.py) | 提供 `_run_commands` 构建 CLI 命令及 `__update_block_run_status` |

---

## 二、入口协作：从触发到 K8s Job 创建

### 2.1 两条执行路径概览

Mage 的 K8s 执行有两种粒度：

```
                    ┌─────────────────────────┐
                    │  PipelineRun (触发)      │
                    └────────────┬────────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
           ▼                     ▼                     ▼
  run_pipeline_in_one_process  按 Block 调度     STREAMING/INTEGRATION
  (整 Pipeline 一个 K8s Job)  (每个 Block 一个 K8s Job)
```

### 2.2 调度入口：PipelineScheduler

调度起点在 [PipelineScheduler.start()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L130-L203)：

```python
def start(self, should_schedule: bool = True) -> bool:
    # 1. 创建 BlockRun 记录
    self.pipeline_run.create_block_runs()
    # 2. PipelineRun 状态 → RUNNING
    self.pipeline_run.update(status=PipelineRun.PipelineRunStatus.RUNNING)
    # 3. 进入调度
    if should_schedule:
        self.schedule()
```

[schedule()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L212-L338) 方法根据 pipeline 配置走不同分支：

```python
def schedule(self, block_runs=None):
    if self.pipeline.run_pipeline_in_one_process:
        self.__schedule_pipeline()      # 路径 A：整 Pipeline 一个进程
    elif PipelineType.STREAMING == self.pipeline.type:
        self.__schedule_pipeline()      # 路径 A：流式管道
    elif PipelineType.INTEGRATION == self.pipeline.type:
        self.__schedule_integration_streams()  # 集成管道特殊处理
    else:
        self.__schedule_blocks(block_runs)     # 路径 B：按 Block 调度
```

### 2.3 路径 A：整 Pipeline 一个 K8s Job

调度逻辑见 [__schedule_pipeline()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L836-L869)：

```python
def __schedule_pipeline(self):
    job_manager.add_job(
        JobType.PIPELINE_RUN,
        self.pipeline_run.id,
        run_pipeline,               # 实际执行函数
        self.pipeline_run.id,       # 参数
        self.pipeline_run.get_variables(),
        self.build_tags(),
    )
```

[run_pipeline()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1336-L1362) 函数是关键桥梁：

```python
def run_pipeline(pipeline_run_id, variables, tags, allow_blocks_to_fail=False):
    pipeline_run = PipelineRun.query.get(pipeline_run_id)
    executor_type = ExecutorFactory.get_pipeline_executor_type(pipeline)
    pipeline_run.update(executor_type=executor_type)
    
    # 如果 executor_type == K8S，这里返回 K8sPipelineExecutor
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

[ExecutorFactory.get_pipeline_executor()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/executor_factory.py#L39-L92) 的分派逻辑：

```python
elif executor_type == ExecutorType.K8S:
    from mage_ai.data_preparation.executors.k8s_pipeline_executor import K8sPipelineExecutor
    return K8sPipelineExecutor(pipeline, execution_partition=execution_partition)
```

### 2.4 路径 B：每个 Block 一个 K8s Job

调度逻辑见 [__schedule_blocks()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L601-L670)：

```python
def __schedule_blocks(self, block_runs=None):
    # 更新 BlockRun 状态（如 UPSTREAM_FAILED、CONDITION_FAILED）
    self.pipeline_run.update_block_run_statuses(self.pipeline_run.initial_block_runs)
    # 获取可执行的 block（依赖已满足）
    block_runs_to_schedule = self.pipeline_run.executable_block_runs(...)
    
    for b in block_runs_to_schedule[:block_run_quota]:
        b.update(status=BlockRun.BlockRunStatus.QUEUED)
        job_manager.add_job(
            JobType.BLOCK_RUN,
            b.id,
            run_block,              # 实际执行函数
            self.pipeline_run.id,
            b.id,
            self.pipeline_run.get_variables(),
            ...
        )
```

[run_block()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1275-L1333) 函数调用执行器：

```python
return ExecutorFactory.get_block_executor(
    pipeline, block_uuid,
    block_run_id=block_run.id,
    execution_partition=execution_partition,
).execute(...)
```

### 2.5 K8sPipelineExecutor.execute() → K8s Job 创建

核心代码在 [K8sPipelineExecutor.execute()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/k8s_pipeline_executor.py#L44-L63)：

```python
def execute(self, pipeline_run_id=None, global_vars=None, **kwargs):
    job_manager = self.get_job_manager(pipeline_run_id=pipeline_run_id, ...)
    cmd = self._run_commands(global_vars=global_vars, pipeline_run_id=pipeline_run_id, **kwargs)
    job_manager.run_job(cmd, k8s_config=self.executor_config)
```

`_run_commands()` 由基类 [PipelineExecutor._run_commands()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/pipeline_executor.py#L181-L215) 提供，构建出实际在 K8s Pod 内执行的 CLI 命令：

```bash
/app/run_app.sh mage run <repo_path> <pipeline_uuid> \
  --executor-type local_python \
  [--execution-partition <partition>] \
  [--pipeline-run-id <id>]
```

关键点：**K8s Job 内部仍然使用 `local_python` 执行器**，即 K8s 只是将执行环境搬到了独立 Pod，Pod 内部是普通的本地执行。

### 2.6 Job 命名规则

Job 名称由 [K8sPipelineExecutor.get_job_manager()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/k8s_pipeline_executor.py#L65-L89) 构建：

```python
job_name = f'{MAGE_CLUSTER_UUID}-{job_name_prefix}-pipeline-{pipeline_run_id}'
# 例如：cluster123-data-prep-pipeline-456
```

Block 级 Job 命名见 [K8sBlockExecutor._execute()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/k8s_block_executor.py#L46-L51)：

```python
job_name = f'{MAGE_CLUSTER_UUID}-{job_name_prefix}-block-{block_run_id}'
```

`job_name_prefix` 支持模板变量 `{trigger_name}`，会被替换为经过 `clean_name` 规范化后的触发名称。

### 2.7 K8s Job 对象构建

[JobManager.create_job_object()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py#L100-L153) 负责构建 K8s Job 规格：

1. **Pod 规格合并**：通过 [merge_pod_spec()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py#L155-L179) 将用户配置的 Pod 规格与当前 Mage Server Pod 的规格合并（继承 volumes、tolerations、node_selector、scheduler_name、image_pull_secrets）。
2. **Container 规格合并**：通过 [merge_container_spec()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py#L181-L204) 继承 Mage Server 容器的 env、env_from、volume_mounts、image。
3. **资源限制**：支持 `resource_limits` 和 `resource_requests`（向后兼容），以及新的 `container_config.resources`。
4. **Job 配置**：`active_deadline_seconds`、`backoff_limit`（默认 0，不重试）、`ttl_seconds_after_finished`。

---

## 三、状态推进：从 K8s Pod 到 PipelineRun/BlockRun

### 3.1 状态枚举

[PipelineRunStatus](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/db/models/schedules.py#L769-L774)：

```
INITIAL → RUNNING → COMPLETED
                    → FAILED
                    → CANCELLED
```

[BlockRunStatus](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1664-L1672)：

```
INITIAL → QUEUED → RUNNING → COMPLETED
                               → FAILED
                               → CANCELLED
                               → UPSTREAM_FAILED
                               → CONDITION_FAILED
```

### 3.2 K8s Job 轮询机制

[JobManager.run_job()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py#L67-L98) 是状态推进的核心：

```python
def run_job(self, command, k8s_config=None):
    if not self.job_exists():
        job = self.create_job_object(command, k8s_config=k8s_config)
        self.create_job(job)  # 调用 K8s API 创建 Job

    job_completed = False
    while not job_completed:
        # 每 5 秒轮询一次 Job 状态
        api_response = self.batch_api_client.read_namespaced_job(
            name=self.job_name, namespace=self.namespace
        )
        # 只要 succeeded 或 failed 有值，说明 Job 已结束
        if api_response.status.succeeded is not None or \
                api_response.status.failed is not None:
            job_completed = True
        time.sleep(5)

    self.delete_job()  # 执行完成后立即删除 Job
    if api_response.status.succeeded is None:
        raise Exception(f'Failed to execute k8s job {self.job_name}')
```

**推进机制图解**：

```
调度线程 (Scheduler)
    │
    ▼
K8sPipelineExecutor.execute() / K8sBlockExecutor._execute()
    │  (同步阻塞调用)
    ▼
JobManager.run_job()
    │
    ├── create_job()  →  K8s API Server  →  Pod 创建
    │
    └── while 循环 (每 5s)
            │
            ├── read_namespaced_job()  ←  K8s API Server
            │
            ├── succeeded != None  →  break, delete_job(), 正常返回
            │
            └── failed != None     →  break, delete_job(), raise Exception
```

**注意**：`run_job()` 是**同步阻塞**的，调用线程会一直等待直到 K8s Job 完成。这意味着：
- Pipeline 级 K8s 执行：调度线程会被阻塞，直到整个 Pipeline 在 Pod 内执行完毕
- Block 级 K8s 执行：每个 Block 的 job 线程会被阻塞，直到对应 Pod 执行完毕

### 3.3 Pod 内部执行与状态回写

K8s Job 创建的 Pod 内部执行 `mage run` CLI 命令，入口在 [cli/main.py:run()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/cli/main.py#L160-L276)。

Pod 内执行使用 `local_python` 执行器，状态通过数据库直接回写：

**Block 状态更新**由 [BlockExecutor.__update_block_run_status()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L1369-L1399) 负责：

```python
def __update_block_run_status(self, status, block_run_id=None, ...):
    block_run = BlockRun.query.get(block_run_id)
    update_kwargs = dict(status=status)
    
    if status == BlockRun.BlockRunStatus.COMPLETED:
        update_kwargs['completed_at'] = datetime.now(tz=pytz.UTC)
        # ... 更新 metrics
    elif status == BlockRun.BlockRunStatus.FAILED:
        # ... 记录 error 到 metrics
    
    block_run.update(**update_kwargs)
```

状态更新触发点在 [BlockExecutor.execute()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/block_executor.py)：
- **RUNNING**：L121-L126，在执行 block 前更新
- **COMPLETED**：L739-L745，在 `_execute()` 成功返回后更新
- **FAILED**：L687-L695，在异常捕获时更新
- **CONDITION_FAILED**：L460-L465，条件判断失败时更新

### 3.4 Pipeline 级状态聚合

Pipeline 状态在调度器的 [schedule()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L240-L331) 中聚合：

```python
if self.pipeline_run.all_blocks_completed(self.allow_blocks_to_fail):
    if self.pipeline_run.any_blocks_failed():
        # 有 Block 失败 → Pipeline FAILED
        self.pipeline_run.update(status=PipelineRun.PipelineRunStatus.FAILED, ...)
        self.on_pipeline_run_failure(error_msg)
    else:
        # 所有 Block 成功 → Pipeline COMPLETED
        self.pipeline_run.complete()  # 设置 status=COMPLETED, completed_at
        self.notification_sender.send_pipeline_run_success_message(...)
```

### 3.5 调度循环与事件驱动

按 Block 调度时，状态推进是事件驱动的：

1. `on_block_complete()` [L377-L410](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L377-L410)：Block 完成时更新状态并触发重新调度
2. `on_block_failure()` [L443-L488](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L443-L488)：Block 失败时更新状态

但对于 K8s 执行器，Block 完成/失败回调通常不在调度器线程内触发（因为执行在远程 Pod），而是通过：
- Pod 内部执行完后直接更新 DB 状态
- 调度器的定时轮询（外部调度循环周期性调用 `schedule()`）发现状态变化

---

## 四、收尾处理：成功、失败、取消

### 4.1 成功路径

```
K8s Job succeeded
    │
    ▼
JobManager.run_job() 返回 (无异常)
    │
    ▼
Pod 内 mage run 正常退出 (exit 0)
    │
    ├── Block Run: __update_block_run_status(COMPLETED)  [Pod 内]
    │       │
    │       ▼
    │   completed_at = now, metrics 更新
    │
    └── Pipeline Run (整 Pipeline K8s 执行):
            调度器 schedule() 检测到 all_blocks_completed
                │
                ▼
            pipeline_run.complete()
                │
                ▼
            发送成功通知 + 使用统计上报
```

### 4.2 失败路径

```
K8s Job failed / Pod 异常退出
    │
    ▼
JobManager.run_job() → raise Exception("Failed to execute k8s job...")
    │
    ├── (整 Pipeline K8s 执行) 异常向上传播，被调度器捕获
    │       │
    │       ▼
    │   on_pipeline_run_failure()
    │       │
    │       ├── pipeline_run.update(status=FAILED)
    │       ├── cancel_block_runs_and_jobs()  [见 4.3]
    │       └── 发送失败通知
    │
    └── (Block 级 K8s 执行) BlockExecutor.execute() 捕获异常
            │
            ▼
        __update_block_run_status(FAILED)
            │
            ├── error 写入 block_run.metrics.error
            └── on_failure 回调 → 调度器 on_block_failure()
```

失败细节存储在 [BlockRun.metrics](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L456-L461)：

```python
metrics['error'] = dict(
    error=str(error.get('error')),      # 异常类型
    errors=error.get('errors'),         # 堆栈帧
    message=error.get('message'),       # 完整 traceback 字符串
)
```

### 4.3 取消路径

取消入口有三个：

#### 入口 1：用户主动取消 PipelineRun

[stop_pipeline_run()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1426-L1457)：

```python
def stop_pipeline_run(pipeline_run, pipeline=None, status=CANCELLED):
    if pipeline_run.status not in [INITIAL, RUNNING]:
        return
    pipeline_run.update(status=status)              # 1. PipelineRun → CANCELLED
    UsageStatisticLogger().pipeline_run_ended_sync(pipeline_run)
    cancel_block_runs_and_jobs(pipeline_run, pipeline)  # 2. 取消 Block 和 Job
```

#### 入口 2：Pipeline 超时

[PipelineScheduler.__check_pipeline_run_timeout()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L528-L558) 检测超时后：

```python
self.pipeline_run.update(status=status)  # FAILED 或自定义超时状态
self.on_pipeline_run_failure('Pipeline run timed out.', status=status)
```

#### 入口 3：Block 超时

[PipelineScheduler.__check_block_run_timeout()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L567-L599)：

```python
if time_difference > int(block.timeout):
    block_executor.logger.error(...)           # 写超时日志
    self.on_block_failure(block_run.block_uuid)  # Block → FAILED
    job_manager.kill_block_run_job(block_run.id)  # 杀进程
```

#### 核心取消逻辑：cancel_block_runs_and_jobs()

[L1460-L1519](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1460-L1519)：

```python
def cancel_block_runs_and_jobs(pipeline_run, pipeline=None):
    # 1. 批量更新 BlockRun 状态 → CANCELLED
    block_runs_to_cancel = [b for b in pipeline_run.block_runs
                            if b.status in [INITIAL, QUEUED, RUNNING]]
    BlockRun.batch_update_status(
        [b.id for b in block_runs_to_cancel],
        BlockRun.BlockRunStatus.CANCELLED
    )

    # 2. 杀 Job（区分不同执行模式）
    if pipeline and (pipeline.type in [INTEGRATION, STREAMING]
                     or pipeline.run_pipeline_in_one_process):
        # 整 Pipeline / 流式 / 集成：杀 PipelineRun Job
        job_manager.kill_pipeline_run_job(pipeline_run.id)
        
        # 关键：K8s 执行器的取消
        if pipeline_run.executor_type == ExecutorType.K8S:
            ExecutorFactory.get_pipeline_executor(
                pipeline,
                executor_type=pipeline_run.executor_type,
            ).cancel(pipeline_run_id=pipeline_run.id)  # ← 调 K8s 删除 Job
    else:
        # 按 Block 调度：逐个杀 BlockRun Job
        for b in running_blocks:
            job_manager.kill_block_run_job(b.id)

    # 3. 异步清理（GenericJob）
    GenericJob.enqueue_cancel_pipeline_run(pipeline_run.id, cancelled_block_run_ids)
```

#### K8s Job 删除实现

[K8sPipelineExecutor.cancel()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/data_preparation/executors/k8s_pipeline_executor.py#L30-L42)：

```python
def cancel(self, pipeline_run_id=None):
    if pipeline_run_id is None:
        return
    try:
        job_manager = self.get_job_manager(pipeline_run_id=pipeline_run_id)
        job_manager.delete_job()
    except Exception:
        traceback.print_exc()
```

[JobManager.delete_job()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py#L231-L241)：

```python
def delete_job(self):
    self.batch_api_client.delete_namespaced_job(
        name=self.job_name,
        namespace=self.namespace,
        body=client.V1DeleteOptions(
            propagation_policy='Foreground',  # 级联删除 Pod
            grace_period_seconds=0             # 立即终止
        )
    )
```

使用 `Foreground` 级联策略 + 0 秒宽限期，确保 K8s Job 和其管理的 Pod 被立即删除。

### 4.4 Job 自动清理

正常执行完成后，[JobManager.run_job()](file:///d:/fz/0601/solo-dogfeeding/code/314-mage-ai/mage_ai/services/k8s/job_manager.py#L95) 会主动调用 `delete_job()`，避免 K8s 中残留大量已完成的 Job。

此外，K8s Job 本身配置了 `ttl_seconds_after_finished`（可通过 `job_config` 配置），K8s 也会在指定时间后自动清理已完成的 Job。

---

## 五、完整链路时序图（Pipeline 级 K8s 执行）

```
用户/触发器    PipelineScheduler    run_pipeline    K8sPipelineExecutor    JobManager      K8s API
    │               │                   │                   │                │              │
    │  start()      │                   │                   │                │              │
    │──────────────>│                   │                   │                │              │
    │               │ create_block_runs │                   │                │              │
    │               │ status=RUNNING    │                   │                │              │
    │               │ schedule()        │                   │                │              │
    │               │ __schedule_pipeline()                │                │              │
    │               │ add_job(run_pipeline)                │                │              │
    │               │──────────────────>│                   │                │              │
    │               │                   │ get_pipeline_executor(K8S)         │              │
    │               │                   │─ ─ ─ ─ ─ ─ ─ ─ ─ >│                │              │
    │               │                   │                   │                │              │
    │               │                   │ execute()         │                │              │
    │               │                   │──────────────────>│                │              │
    │               │                   │                   │ get_job_manager()            │
    │               │                   │                   │ _run_commands()              │
    │               │                   │                   │ run_job(cmd)   │              │
    │               │                   │                   │───────────────>│              │
    │               │                   │                   │                │ create Job   │
    │               │                   │                   │                │─────────────>│
    │               │                   │                   │                │  (Pod 创建)  │
    │               │                   │                   │  while 轮询     │              │
    │               │                   │                   │  (每 5s)        │              │
    │               │                   │                   │  read_job()────│─────────────>│
    │               │                   │                   │                │<─────────────│
    │               │                   │                   │  ... (重复)    │              │
    │               │                   │                   │                │              │
    │               │                   │   (Pod 内 mage run 执行)            │              │
    │               │                   │   ┌──────────────────────────────────────────┐  │
    │               │                   │   │  BlockExecutor.execute()                │  │
    │               │                   │   │    → status=RUNNING                     │  │
    │               │                   │   │    → _execute() (实际业务逻辑)           │  │
    │               │                   │   │    → status=COMPLETED / FAILED          │  │
    │               │                   │   │  (全部通过 DB 回写)                     │  │
    │               │                   │   └──────────────────────────────────────────┘  │
    │               │                   │                   │                │              │
    │               │                   │                   │ succeeded=1    │<─────────────│
    │               │                   │                   │ delete_job()───│─────────────>│
    │               │                   │                   │ (正常返回)     │              │
    │               │                   │<──────────────────│                │              │
    │               │ schedule() 检测 all_blocks_completed │                │              │
    │               │ pipeline_run.complete()               │                │              │
    │               │ (发送通知 + 统计上报)                  │                │              │
```

---

## 六、关键设计要点总结

### 6.1 两层执行器模型

```
调度层 Executor (K8sPipelineExecutor / K8sBlockExecutor)
    │  负责：启动 K8s Job、轮询状态、取消 Job
    ▼
Pod 内 Executor (PipelineExecutor / BlockExecutor, local_python)
       负责：实际执行业务逻辑、更新 DB 状态
```

调度层执行器**只关心 K8s Job 的生命周期**，不介入具体 Block 的执行细节；Pod 内执行器通过数据库与调度层解耦。

### 6.2 配置合并策略

K8s Job 的 Pod/Container 配置采用「用户配置 + Mage Server Pod 继承」的混合策略：
- 用户自定义优先级更高（非 None 值覆盖）
- 列表类字段（env、volume_mounts、volumes）取并集去重
- 继承 Mage Server Pod 的 image、环境变量、存储卷挂载，确保执行环境一致

### 6.3 同步阻塞模型

`JobManager.run_job()` 的 while 轮询是同步阻塞的，调用方线程会被占用直到 K8s Job 完成。这简化了编程模型，但要求调度层有足够的并发能力（由 `job_manager` 的线程池承担）。

### 6.4 状态最终一致性

- **Block 状态**：由 Pod 内执行器直接写入 DB，保证准确
- **Pipeline 状态**：由调度器周期扫描 BlockRun 状态后聚合
- **K8s Job 状态**：由 JobManager 轮询并决定阻塞调用是否返回

三者通过 DB + K8s API 实现最终一致，没有分布式事务。

### 6.5 取消的幂等性

各取消入口（`stop_pipeline_run`、超时检测、`cancel` API）都会先检查状态是否仍在 `[INITIAL, RUNNING]`，避免重复取消造成副作用。`JobManager.delete_job()` 也有 try-except 保护，容忍 Job 已不存在的情况。

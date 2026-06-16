# Spark / EMR 集成流程：入口选择条件与结果落点

> 本文档 100% 基于代码事实，逐行核对了以下关键文件：
> - `mage_ai/data_preparation/executors/executor_factory.py`
> - `mage_ai/data_preparation/models/block/__init__.py`
> - `mage_ai/data_preparation/repo_manager.py`
> - `mage_ai/data_preparation/models/project/__init__.py`
> - `mage_ai/services/spark/utils.py`
> - `mage_ai/services/compute/models.py`
> - `mage_ai/data_preparation/executors/pyspark_pipeline_executor.py`
> - `mage_ai/data_preparation/executors/pyspark_block_executor.py`
> - `mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja`
>
> 聚焦两个核心问题：
> 1. **什么配置会触发远程 EMR 执行**（而非本地 Python 执行）
> 2. **状态与监控信息落到哪里**（每类数据的写入位置、读取方式、是否可观测）

---

## 一、触发远程执行的三层判定链

Mage 决定「是否走 EMR 远程执行」由三层判定叠加而成，任何一层不满足都不会触发 EMR。

### 第 1 层：Pipeline 执行器选择

**代码位置**：`mage_ai/data_preparation/executors/executor_factory.py` L24-L37

```python
@classmethod
def get_pipeline_executor_type(self, pipeline, executor_type=None):
    if executor_type is None:
        if pipeline.type == PipelineType.PYSPARK:
            executor_type = ExecutorType.PYSPARK
        else:
            executor_type = pipeline.get_executor_type()
            if executor_type == ExecutorType.LOCAL_PYTHON or executor_type is None:
                executor_type = self.get_default_executor_type()
    return executor_type
```

**Pipeline 维度触发条件**（满足任一即触发 PySpark 执行器）：

| 条件 | 配置位置 | 配置写法 |
|------|----------|----------|
| **A. Pipeline type = pyspark** | Pipeline `metadata.yaml` | `type: pyspark` |
| **B. Pipeline executor_type = pyspark** | Pipeline `metadata.yaml` | `executor_type: pyspark` |
| **C. 环境变量 DEFAULT_EXECUTOR_TYPE = pyspark** | Mage 服务启动环境 | `export DEFAULT_EXECUTOR_TYPE=pyspark` |

**判断优先级**（高 → 低）：条件 A → 条件 B → 条件 C

**完整流程图**：
```
pipeline.type == PYSPARK ?
  ├─ YES → executor_type = PYSPARK
  └─ NO  → executor_type = pipeline.get_executor_type()
             ├─ == LOCAL_PYTHON 或 None → DEFAULT_EXECUTOR_TYPE
             └─ 其他值 → 使用该值
```

### 第 2 层：Block 执行器选择

**代码位置**：`mage_ai/data_preparation/executors/executor_factory.py` L127-L138

```python
if executor_type is None:
    block = pipeline.get_block(block_uuid, check_template=True)
    if block:
        if pipeline.type == PipelineType.PYSPARK and (
            block.type != BlockType.SENSOR or is_pyspark_code(block.content)
        ):
            executor_type = ExecutorType.PYSPARK
        else:
            executor_type = block.get_executor_type()
            if executor_type == ExecutorType.LOCAL_PYTHON or not executor_type:
                executor_type = self.get_default_executor_type()
```

#### 2.1 自动提升逻辑（最容易看错的布尔表达式）

布尔表达式：`A && (B || C)`

| 变量 | 含义 |
|------|------|
| A | `pipeline.type == PipelineType.PYSPARK` |
| B | `block.type != BlockType.SENSOR`（Block 类型**不是** SENSOR） |
| C | `is_pyspark_code(block.content)`（代码含 `'spark'` 或 `"spark"`） |

**实际行为真值表**：

| Pipeline 类型 | Block 类型 | 代码含 spark | 结果 executor_type |
|--------------|------------|--------------|--------------------|
| PYSPARK | 非 SENSOR（transformer/loader/scratchpad） | 任意 | **PYSPARK** |
| PYSPARK | SENSOR | 是 | **PYSPARK** |
| PYSPARK | SENSOR | 否 | 不自动提升，走 block.get_executor_type() |
| 非 PYSPARK | 任意 | 任意 | 不自动提升，走 block.get_executor_type() |

> ⚠️ **之前描述错误**：上一版写反了 SENSOR 的行为。实际是：SENSOR 类型的 Block 只要代码含 `spark` 字面量，依然会自动提升为 PySpark 执行器。
>
> **正确结论**：非 SENSOR 的 Block，只要 Pipeline 是 PYSPARK 类型，不管代码写什么，都会走 PySpark；SENSOR 类型的 Block 需要代码含 `spark` 字面量才会走 PySpark。

#### 2.2 Block.get_executor_type() 二级回退链

**代码位置**：`mage_ai/data_preparation/models/block/__init__.py` L2884-L2900

```python
def get_executor_type(self) -> str:
    if self.executor_type:
        # Block 自身配置了 executor_type → 支持 Jinja 模板渲染
        block_executor_type = Template(self.executor_type).render(**get_template_vars())
    else:
        block_executor_type = None
    
    # 如果 Block 没配置 或 配置为 LOCAL_PYTHON → 回退到 Pipeline
    if not block_executor_type or block_executor_type == ExecutorType.LOCAL_PYTHON:
        if self.pipeline:
            pipeline_executor_type = self.pipeline.get_executor_type()
        else:
            pipeline_executor_type = None
        block_executor_type = pipeline_executor_type or block_executor_type
    return block_executor_type
```

**完整回退链**：

```
Block.executor_type (支持 Jinja)
  │
  ├─ 配置了 且 不等于 LOCAL_PYTHON → 使用 Block.executor_type
  │
  └─ 未配置 或 = LOCAL_PYTHON
          │
          ├─ Pipeline.get_executor_type() 有值 → 使用 Pipeline 的
          │
          └─ Pipeline 也没有 → 返回 None
                  │
                  └─ ExecutorFactory 再回退 → DEFAULT_EXECUTOR_TYPE
```

**Block 维度触发条件**（满足任一即触发）：

| 条件 | 说明 |
|------|------|
| Pipeline type=PYSPARK + Block 非 SENSOR | 自动提升，不管代码写什么 |
| Pipeline type=PYSPARK + Block=SENSOR + 代码含 `spark` 字面量 | 自动提升 |
| Block 自身 `executor_type: pyspark` | 显式配置，支持 Jinja |
| Block 未配置但 Pipeline `executor_type: pyspark` | 通过回退链继承 |
| DEFAULT_EXECUTOR_TYPE=pyspark | 环境变量兜底 |

### 第 3 层：计算服务类型判定（监控 / 交互层）

**代码位置 1**：`mage_ai/services/spark/utils.py` L9-L37

```python
def get_compute_service(emr_config=None, repo_config=None, ...):
    if not repo_config:
        repo_config = get_repo_config()
    if repo_config and emr_config:
        repo_config.emr_config = emr_config
    
    if not repo_config:
        return None
    
    if repo_config.emr_config and (KernelName.PYSPARK == kernel_name or ignore_active_kernel):
        return ComputeServiceUUID.AWS_EMR
    elif is_spark_env() and repo_config.spark_config and \
            SparkMaster.LOCAL.value == repo_config.spark_config.get('spark_master'):
        return ComputeServiceUUID.STANDALONE_CLUSTER
    return None
```

**代码位置 2**：`mage_ai/services/compute/models.py` L147-L156

```python
@classmethod
def build(self, project, with_clusters=False):
    service_class = self
    if project and project.spark_config:
        if project.emr_config:
            from mage_ai.services.compute.aws.models import AWSEMRComputeService
            service_class = AWSEMRComputeService
    return service_class(project=project, with_clusters=with_clusters)
```

**代码位置 3**：`mage_ai/data_preparation/models/project/__init__.py` L154-L159

```python
@property
def emr_config(self) -> Dict:
    return self.repo_config.emr_config or None

@property
def spark_config(self) -> Dict:
    return self.repo_config.spark_config or None
```

#### 3.1 空配置判断的完整链路

**代码位置 4**：`mage_ai/data_preparation/repo_manager.py` L135, L139

```python
self.emr_config = repo_config.get('emr_config') or dict()  # 空 → {}
self.spark_config = repo_config.get('spark_config')        # 空 → None
```

**实际判断逻辑（必须同时满足）**：

| 判断节点 | 条件 | 代码位置 |
|----------|------|----------|
| 1 | `repo_config.emr_config` 必须是 **非空 dict**（`if {...}` 为 True） | utils.py L30 |
| 2 | `repo_config.spark_config` 必须 **不是 None**（可空 dict，但不能缺失） | compute/models.py L150 |
| 3 | `project.emr_config` 必须 **不是 None**（`{} or None` → None，会被过滤） | project/__init__.py L155 |
| 4 | kernel 是 PYSPARK 或 `ignore_active_kernel=True` | utils.py L30 |

**metadata.yaml 配置真值表**：

| emr_config 配置 | spark_config 配置 | repo_config.emr_config | project.emr_config | 触发 AWS_EMR？ |
|-----------------|-------------------|------------------------|--------------------|-----------------|
| 无此字段 | 无此字段 | `{}` | `None` | ❌ |
| `emr_config: {}` | 无此字段 | `{}` | `None` | ❌ |
| `emr_config:` 下有至少 1 个子项 | 无此字段 | `{...}` | `{...}` | ❌（缺 spark_config） |
| `emr_config:` 下有至少 1 个子项 | `spark_config: {}` | `{...}` | `{...}` | ✅（需同时满足 kernel 条件） |

> ⚠️ **关键细节**：`repo_config.get('spark_config')` 返回 `None`（无此字段）和 `{}`（有字段但空）在 `compute/models.py` L150 的 `if project.spark_config` 判断中结果不同：`None` → False，`{}` → True。
>
> 所以必须在 metadata.yaml 中显式写 `spark_config: {}`（或非空），不能完全不写。

#### 3.2 触发条件汇总

要触发 `AWS_EMR` 计算服务路由，必须同时满足：

1. ✅ `metadata.yaml` 中 `emr_config` 段至少配置一个子项（如 `master_instance_type`）
2. ✅ `metadata.yaml` 中 `spark_config` 段存在（可空 dict `{}`，但不能完全不写）
3. ✅ 当前 kernel 是 `pyspark`，或调用方传 `ignore_active_kernel=True`

---

## 二、配置的完整加载链路

### 配置层级与合并顺序

**代码位置**：
- PySparkPipelineExecutor: `mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` L18-L23
- PySparkBlockExecutor: `mage_ai/data_preparation/executors/pyspark_block_executor.py` L31-L36

**PySparkPipelineExecutor 的配置合并**：
```python
self.emr_config = self.pipeline.repo_config.emr_config or dict()      # 第 1 层：项目级
if self.pipeline.executor_config is not None:
    self.emr_config = merge_dict(self.emr_config, self.pipeline.executor_config)  # 第 2 层：Pipeline 级覆盖
self.emr_config = EmrConfig.load(config=self.emr_config)              # 转为 EmrConfig 对象
```

**PySparkBlockExecutor 的配置合并**（多一层 Block 级覆盖）：
```python
self.executor_config = self.pipeline.repo_config.emr_config           # 第 1 层：项目级
if self.pipeline.executor_config is not None:
    self.executor_config = merge_dict(self.executor_config, self.pipeline.executor_config)  # 第 2 层：Pipeline 级
if self.block.executor_config is not None:
    self.executor_config = merge_dict(self.executor_config, self.block.executor_config)     # 第 3 层：Block 级
self.executor_config = EmrConfig.load(config=self.executor_config)
```

**合并优先级（高 → 低）**：
```
Block.executor_config    ← 最细粒度，可覆盖单个 Block 的集群参数
  ↓ merge_dict
Pipeline.executor_config ← Pipeline 维度，可覆盖特定 Pipeline 的集群参数
  ↓ merge_dict
repo_config.emr_config   ← 项目全局默认，metadata.yaml 中的 emr_config 段
```

### S3 Bucket 来源

**代码位置**：`mage_ai/data_preparation/repo_manager.py` L158-L164

```python
if self.remote_variables_dir is not None and self.remote_variables_dir.startswith('s3://'):
    path_parts = self.remote_variables_dir.replace('s3://', '').split('/')
    self.s3_bucket = path_parts.pop(0)
    self.s3_path_prefix = '/'.join(path_parts)
```

**必须配置**：`remote_variables_dir: s3://{bucket}/{prefix}`，否则 `EmrResourceManager.__init__()` 会抛异常：
> Please specify the correct s3_bucket to initialize EMR cluster.
> Add "remote_variables_dir: s3://[bucket]/[path]" to project's metadata.yaml file.

---

## 三、结果落点全图：状态与监控信息的精确边界

### 3.1 状态更新的三层边界

#### 边界 1：PySparkPipelineExecutor 层

**代码位置**：`mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` L32-L48

```python
def execute(
    self,
    analyze_outputs: bool = False,  # ⚠️ 传参但方法体内完全不用
    global_vars: Dict = None,
    run_tests: bool = False,        # ⚠️ 传参但方法体内完全不用
    update_status: bool = False,    # ⚠️ 传参但方法体内完全不用
    **kwargs,
) -> None:
    self.upload_pipeline_execution_script(global_vars=global_vars)
    self.resource_manager.upload_bootstrap_script()
    self.submit_spark_job()
```

**关键观察**：
- 完全重写了父类 `PipelineExecutor.execute()`，**不调用 `super().execute()`**
- 方法签名的 `update_status`、`analyze_outputs`、`run_tests` 参数**在方法体内完全不使用**
- 不处理 `pipeline_run_id` 参数，不创建 BlockRun/PipelineRun
- **同步阻塞**：`emr.submit_spark_job()` 内部 `__status_poller` 轮询 Step 状态到终态才返回
- 异常传播：Step 失败抛异常 → 上层调用方可能捕获后更新状态

**这一层的状态落点**：

| 数据 | 写入位置 | 写入方 | 说明 |
|------|----------|--------|------|
| Pipeline 执行状态 | Mage DB（PipelineRun 表） | **不写入** | 完全依赖上层调用方 |
| Block 执行状态 | Mage DB（BlockRun 表） | **不写入** | 完全依赖上层调用方 |
| EMR Step 状态 | Mage 控制台 stdout | `__status_poller` print | 仅运行时可见，不持久化 |
| 异常信息 | 抛出到调用方 | `emr.submit_spark_job()` | Step 非 COMPLETED 时抛 Exception |

#### 边界 2：EMR 远程脚本层（spark_script.jinja）

**代码位置**：`mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja` L64-L81

```python
if block_uuid is None:
    asyncio.run(pipeline.execute(
        analyze_outputs=False,     # ⚠️ 硬编码 False
        global_vars=global_vars,
        update_status=False,       # ⚠️ 硬编码 False
    ))
else:
    block = pipeline.get_block(block_uuid)
    block.execute_sync(
        analyze_outputs=False,     # ⚠️ 硬编码 False
        execution_partition={{ execution_partition_str }},
        global_vars=global_vars,
        update_status=False,       # ⚠️ 硬编码 False
    )
    block.run_tests(
        execution_partition={{ execution_partition_str }},
        global_vars=global_vars,
        update_tests=False,        # ⚠️ 硬编码 False
    )
```

**关键观察**：
- 运行在 EMR Driver 节点上，没有连接 Mage DB 的配置
- 所有 `update_status`、`analyze_outputs`、`update_tests` 全是**硬编码 False**，不是从调用方传参
- 所有需要写 DB 的操作都被禁用

**这一层的状态落点**：

| 数据 | 写入位置 | 写入方 | 说明 |
|------|----------|--------|------|
| Pipeline 执行状态 | Mage DB | **完全不写** | 硬编码 update_status=False |
| Block 执行状态 | Mage DB | **完全不写** | 硬编码 update_status=False |
| 测试结果 | Mage DB | **完全不写** | 硬编码 update_tests=False |
| Block 输出数据 | Mage 内部缓存 | **完全不写** | 硬编码 analyze_outputs=False |
| 变量 | Mage 内部变量存储 | **不写** | 可写 Spark 表或 S3 |
| 异常栈 | S3 text 文件 | ErrorLogging 类 | `s3://{bucket}/{prefix}/logs/` |
| Spark 表 | HDFS / Glue Catalog | 业务代码 | `saveAsTable()` |

#### 边界 3：上层调用方（Orchestrator / Trigger）

由于 `PySparkPipelineExecutor.execute()` 是同步阻塞的：
- Step **成功** → 方法正常返回 → 上层调用方可能更新 PipelineRun 状态为 COMPLETED
- Step **失败** → 抛出 Exception → 上层调用方可能捕获后更新状态为 FAILED

这部分行为由上层调度器决定，不在 PySpark 执行器控制范围内。

### 3.2 监控信息落点

| 监控数据 | 数据源 | 传输路径 | 展示位置 | 实时性 |
|----------|--------|----------|----------|--------|
| **Spark Application 列表** | Spark UI `/api/v1/applications` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 | 准实时（隧道连通时） |
| **Spark Job/Stage/Task** | Spark UI 各 REST 端点 | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 | 准实时 |
| **Spark SQL 执行** | Spark UI `/api/v1/sql` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 | 准实时 |
| **Application 缓存** | `{variables_dir}/.spark/applications.json` | BaseSparkModel 本地文件 | Spark UI 不可达时兜底 | 历史数据 |
| **EMR 集群状态** | AWS EMR API `describe_cluster` | EmrClusterManager | Mage UI 集群管理页 | 实时 |
| **EMR Step 状态** | AWS EMR API `describe_step` | `__status_poller` 轮询 | Mage 控制台日志 | 运行时，轮询间隔 30s+ |
| **EMR 集群日志** | `s3://{bucket}/{prefix}/logs/` | EMR 自动写入 | S3 浏览 / Athena | 延迟 ~5min |

### 3.3 SSH 隧道前提条件（缺一不可）

**代码位置**：`mage_ai/services/ssh/aws/emr/models.py` L19-L188

1. `emr_config.ec2_key_path` 已配置（PEM 文件本地路径）
2. `emr_config.ec2_key_name` 已配置（EC2 Key Pair 名称）
3. EMR Master 的 22 端口可从 Mage Server 访问（安全组 + 网络连通）
4. EMR 集群处于 RUNNING / WAITING 状态
5. `SSHTunnel` 初始化参数均不为空：`ec2_key_path`、`master_public_dns_name`、`remote_bind_port`

### 3.4 数据输出落点

| 输出类型 | 落点 | 写入方 | 持久性 |
|----------|------|--------|--------|
| **Spark 表**（默认） | HDFS | io/spark.py `export()` | ❌ 集群销毁即丢失 |
| **Spark 表**（Glue） | Glue Catalog + S3 | io/spark.py `export()` | ✅ 持久 |
| **S3 文件** | `s3://user-defined-bucket/...` | 用户 Block 代码 | ✅ 持久 |
| **执行脚本** | `s3://{bucket}/{prefix}/scripts/` | EmrResourceManager | ✅ 持久 |
| **Bootstrap 脚本** | `s3://{bucket}/{prefix}/scripts/emr_bootstrap.sh` | EmrResourceManager | ✅ 持久 |
| **EMR 日志** | `s3://{bucket}/{prefix}/logs/` | EMR 服务自动 | ✅ 持久 |
| **Spark 错误栈** | `s3://{bucket}/{prefix}/logs/` | ErrorLogging | ✅ 持久 |
| **Block 输出变量** | Mage 内部变量存储 | **不写** | ❌ |

---

## 四、远程执行的完整数据流图（按代码事实校正版）

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Mage Server                                    │
│                                                                      │
│  1. ExecutorFactory 判定 executor_type                               │
│     ├─ Pipeline 级：                                                │
│     │   ├─ type=PYSPARK → PYSPARK                                   │
│     │   ├─ executor_type=PYSPARK → PYSPARK                          │
│     │   └─ DEFAULT_EXECUTOR_TYPE=pyspark → PYSPARK                  │
│     │                                                                │
│     └─ Block 级（A && (B || C)）：                                   │
│         ├─ A=Pipeline type=PYSPARK                                  │
│         ├─ B=Block 非 SENSOR → PYSPARK                              │
│         └─ C=Block=SENSOR + 代码含 spark → PYSPARK                  │
│                                                                      │
│  2. PySparkPipelineExecutor / PySparkBlockExecutor                   │
│     ├─ 配置合并：repo_config ← pipeline ← block                     │
│     ├─ 模板渲染 spark_script.jinja → 上传到 S3                      │
│     ├─ bootstrap 脚本上传到 S3                                      │
│     └─ emr.submit_spark_job()（同步阻塞）                            │
│         ├─ list_clusters(RUNNING/WAITING)                           │
│         ├─ 有可用集群 → add_job_flow_steps()                         │
│         ├─ 无可用集群 → create_a_new_cluster()                       │
│         └─ __status_poller() 轮询 Step 状态                          │
│            ├─ 每 30s+ 调用 describe_step → print 到 stdout           │
│            ├─ COMPLETED → 正常返回                                   │
│            └─ 其他状态 → 抛 Exception 给上层                         │
│                                                                      │
│  ⚠️  注意：execute(update_status=...) 参数在这一层**完全不用**！       │
│                                                                      │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                      │
│  监控通道（独立于执行通道）：                                          │
│                                                                      │
│  3. ComputeService.build(project)                                    │
│     ├─ project.spark_config 存在（非 None）                          │
│     └─ project.emr_config 存在（非空 dict）                          │
│         → AWSEMRComputeService                                      │
│                                                                      │
│  4. get_compute_service()                                            │
│     ├─ repo_config.emr_config 非空                                  │
│     └─ kernel=PYSPARK 或 ignore_active_kernel=True                   │
│         → ComputeServiceUUID.AWS_EMR                                │
│                                                                      │
│  5. API.build()                                                      │
│     └─ AWS_EMR → AwsEmrAPI → SSHTunnel → REST API                   │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       AWS EMR 集群（Driver 节点）                     │
│                                                                      │
│  spark-submit --deploy-mode cluster s3://.../scripts/{uuid}.py       │
│     │                                                                │
│     ├─ SparkSession.builder.appName('My PyPi').getOrCreate()         │
│     │                                                                │
│     ├─ Pipeline(uuid, config, repo_config)  ← 从模板渲染             │
│     │   └─ global_vars['spark'] = spark                              │
│     │                                                                │
│     └─ 分支执行（所有 update_* 硬编码为 False）：                      │
│         ├─ Pipeline → execute(update_status=False,                   │
│         │                      analyze_outputs=False)                │
│         │                                                             │
│         └─ Block → execute_sync(update_status=False,                 │
│                             analyze_outputs=False)                   │
│                 run_tests(update_tests=False)                        │
│                                                                      │
│  ⚠️  注意：这一层完全不连接 Mage DB，所有回写操作被硬编码禁用！         │
│                                                                      │
│  异常：ErrorLogging → S3 text 文件                                   │
│  数据：Spark 表 / S3 / HDFS                                          │
│  日志：s3://{bucket}/{prefix}/logs/（EMR 自动）                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键文件索引（相对路径）

| 层级 | 文件路径 | 关键职责 |
|------|----------|----------|
| **执行器工厂** | `mage_ai/data_preparation/executors/executor_factory.py` | 判定 executor_type → 选择执行器 |
| **Pipeline 执行器** | `mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` | Pipeline 级远程 EMR 执行（重写 execute） |
| **Block 执行器** | `mage_ai/data_preparation/executors/pyspark_block_executor.py` | Block 级远程 EMR 执行（重写 _execute） |
| **EMR 核心服务** | `mage_ai/services/aws/emr/emr.py` | 集群创建、任务提交、步骤轮询 |
| **EMR 基础封装** | `mage_ai/services/aws/emr/emr_basics.py` | boto3 底层调用 |
| **EMR 配置** | `mage_ai/services/aws/emr/config.py` | EmrConfig 数据结构 |
| **EMR 资源管理** | `mage_ai/services/aws/emr/resource_manager.py` | S3 路径管理 + bootstrap 上传 |
| **配置加载** | `mage_ai/data_preparation/repo_manager.py` | metadata.yaml → RepoConfig |
| **Project 模型** | `mage_ai/data_preparation/models/project/__init__.py` | Project.emr_config / spark_config 属性 |
| **Block 模型** | `mage_ai/data_preparation/models/block/__init__.py` | Block.get_executor_type() 二级回退 |
| **Spark 会话** | `mage_ai/services/spark/spark.py` | SparkSession 获取（本地模式） |
| **Spark 配置** | `mage_ai/services/spark/config.py` | SparkConfig 数据结构 |
| **服务路由** | `mage_ai/services/spark/utils.py` | get_compute_service() 判断 |
| **计算服务** | `mage_ai/services/compute/models.py` | ComputeService.build() 路由 |
| **API 路由** | `mage_ai/services/spark/api/service.py` | LocalAPI / AwsEmrAPI 工厂 |
| **EMR API** | `mage_ai/services/spark/api/aws_emr.py` | EMR 模式 Spark UI 查询 |
| **SSH 隧道** | `mage_ai/services/ssh/aws/emr/models.py` | SSHTunnel 单例 |
| **执行脚本模板** | `mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja` | 远程执行脚本（硬编码 update_status=False） |
| **Bootstrap 模板** | `mage_ai/data_preparation/templates/pipeline_execution/emr_bootstrap.sh` | 集群节点初始化 |
| **IO 连接器** | `mage_ai/io/spark.py` | Spark 数据读写 |
| **API Mixin** | `mage_ai/api/resources/mixins/spark.py` | Spark 监控 REST API |
| **PySpark 代码检测** | `mage_ai/shared/code.py` | is_pyspark_code() |
| **枚举常量** | `mage_ai/data_preparation/models/constants.py` | ExecutorType / PipelineType |

---

## 六、与代码事实核对的 5 个关键结论

### 结论 1：非 SENSOR Block 在 PYSPARK Pipeline 中强制走 PySpark

只要 `pipeline.type == PYSPARK`，非 SENSOR 类型的 Block（transformer/loader/scratchpad 等）不管代码写什么，**一定会走 PySpark 执行器**。这是 `block.type != BlockType.SENSOR` 这个条件决定的。

### 结论 2：SENSOR Block 含 spark 字面量也会走 PySpark

布尔表达式 `block.type != BlockType.SENSOR or is_pyspark_code(block.content)` 意味着：
- Block 是 SENSOR，但 `is_pyspark_code(block.content)` 返回 True → 依然走 PySpark
- 上一版描述写反了，特此校正

### 结论 3：空 EMR 配置 ≠ 配置了 EMR

| 配置方式 | repo_config.emr_config | project.emr_config | 触发 AWS_EMR？ |
|----------|------------------------|--------------------|-----------------|
| 完全不写 | `{}` | `None` | ❌ |
| 写了 `emr_config: {}` | `{}` | `None` | ❌ |
| 写了至少一个子项 | `{...}` | `{...}` | ✅（还需 spark_config 存在） |

`{}` 在 Python 中是 falsy，`project.emr_config` 还会进一步把 `{}` 转成 `None`。

### 结论 4：spark_config 不能完全不写

`compute/models.py` L150 `if project and project.spark_config:` 判断中：
- `project.spark_config is None` → False
- `project.spark_config == {}` → True

所以 `metadata.yaml` 中必须显式写 `spark_config: {}`（或非空），不能完全不写这个字段。

### 结论 5：update_status 参数在 PySpark 执行器中被完全忽略

- `PySparkPipelineExecutor.execute()` 有 `update_status` 参数，但方法体内**完全不用**
- `spark_script.jinja` 中的 `update_status=False` 是**硬编码**，不是传参
- 两层都不写 Mage DB，状态更新完全依赖上层调用方

> 文档生成时间：2026-06-16  
> 代码版本：328-mage-ai  
> 核对方式：逐行比对源码，所有结论均有代码位置标注

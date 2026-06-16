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

## 一、两条独立路径：远程执行 vs 监控路由

Mage 中「Spark/EMR」涉及两条**完全独立**的代码路径，它们的入口、判断条件、依赖配置各不相同，不能混为一谈：

| 维度 | 路径 A：远程 EMR 执行 | 路径 B：Spark UI 监控路由 |
|------|----------------------|--------------------------|
| **做什么** | 向 EMR 集群提交 spark-submit Step | 决定前端 Spark UI 连本地端口还是 SSH 隧道 |
| **入口代码** | `PySparkPipelineExecutor.execute()` | `get_compute_service()` / `ComputeService.build()` |
| **触发条件** | executor_type == PYSPARK | emr_config + spark_config + kernel |
| **依赖 emr_config** | ✅ 用来建集群/加 Step | ✅ 用来判断走 EMR 路由 |
| **依赖 spark_config** | ❌ **完全不检查** | ✅ `ComputeService.build()` 要求非空 |
| **依赖 kernel 类型** | ❌ **完全不检查** | ✅ `get_compute_service()` 要求 PYSPARK |
| **依赖 s3_bucket** | ✅ 没有则初始化直接报错 | ❌ 不涉及 |

> ⚠️ **最易混淆的点**：远程执行和监控路由可以独立生效。一个 Pipeline 可能已经在 EMR 上跑起来了（路径 A），但 Mage 前端的 Spark UI 页面还是显示空白（路径 B 不通）。反之亦然。

---

## 二、路径 A：远程 EMR 执行的触发条件

### 步骤 1：ExecutorFactory 选出 PySpark 执行器

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

### 步骤 2：Block 级执行器选择

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

#### 2.2 Block.get_executor_type() 二级回退链

**代码位置**：`mage_ai/data_preparation/models/block/__init__.py` L2884-L2900

```python
def get_executor_type(self) -> str:
    if self.executor_type:
        block_executor_type = Template(self.executor_type).render(**get_template_vars())
    else:
        block_executor_type = None
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

### 步骤 3：PySpark 执行器初始化（路径 A 的实际前置约束）

executor_type == PYSPARK 后，`PySparkPipelineExecutor.__init__()` 或 `PySparkBlockExecutor.__init__()` 被调用。它们**不检查 emr_config 是否为空**，也不检查 spark_config 和 kernel 类型，但有唯一前置约束：

**代码位置**：`mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` L18-L28

```python
self.emr_config = self.pipeline.repo_config.emr_config or dict()  # 空 dict 也能跑，用默认值
# ...
self.resource_manager = EmrResourceManager(
    pipeline.repo_config.s3_bucket,        # ← 这个不能是 None
    pipeline.repo_config.s3_path_prefix,
)
```

**代码位置**：`mage_ai/services/aws/emr/resource_manager.py` L16-L20

```python
if self.s3_bucket is None:
    raise Exception('Please specify the correct s3_bucket to initialize EMR cluster.'
                    'Add "remote_variables_dir: s3://[bucket]/[path]" to'
                    ' project\'s metadata.yaml file.')
```

**路径 A 的完整前置条件**：

| 条件 | 代码位置 | 缺失后果 |
|------|----------|----------|
| executor_type == PYSPARK | executor_factory.py | 走 BlockExecutor 本地执行 |
| `remote_variables_dir: s3://bucket/path` | repo_manager.py L158, resource_manager.py L16 | `EmrResourceManager.__init__()` 抛异常 |

**路径 A 不检查的东西**：

| 不检查项 | 说明 |
|----------|------|
| emr_config 是否有内容 | 空 dict `{}` 也行，EmrConfig.load() 会用默认值（如 `r5.4xlarge`） |
| spark_config | 执行器完全不读 spark_config |
| kernel 类型 | 执行器完全不关心 kernel |
| ComputeService 路由 | 执行路径和监控路由完全独立 |

### 步骤 4：提交 EMR Step（路径 A 的执行动作）

**代码位置**：`mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` L46-L48

```python
def execute(self, ...) -> None:
    self.upload_pipeline_execution_script(global_vars=global_vars)  # 渲染 jinja → S3
    self.resource_manager.upload_bootstrap_script()                  # bootstrap → S3
    self.submit_spark_job()                                          # emr.submit_spark_job()
```

`emr.submit_spark_job()` 内部会自动查找/创建 EMR 集群，不需要预先配置。

---

## 三、路径 B：Spark UI 监控路由的触发条件

路径 B 决定 Mage 前端的 Spark UI 页面连接到哪里。有两个独立入口，判断条件不同。

### 入口 B1：get_compute_service()

**代码位置**：`mage_ai/services/spark/utils.py` L9-L37

**调用方**：`mage_ai/services/spark/api/service.py` L17（`API.build()` 内部）

```python
def get_compute_service(emr_config=None, repo_config=None, ...):
    if not repo_config:
        repo_config = get_repo_config()
    if repo_config and emr_config:
        repo_config.emr_config = emr_config
    if not repo_config:
        return None
    if not kernel_name:
        kernel_name = get_active_kernel_name()

    if repo_config.emr_config and (KernelName.PYSPARK == kernel_name or ignore_active_kernel):
        return ComputeServiceUUID.AWS_EMR
    elif is_spark_env() and repo_config.spark_config and \
            SparkMaster.LOCAL.value == repo_config.spark_config.get('spark_master'):
        return ComputeServiceUUID.STANDALONE_CLUSTER
    return None
```

**入口 B1 的判断条件**：

| 条件 | 说明 |
|------|------|
| `repo_config.emr_config` 非空 dict | 直接读 repo_config，不经 project 属性；`{}` 是 falsy |
| kernel == PYSPARK **或** `ignore_active_kernel=True` | `API.build()` 调用时固定传 `ignore_active_kernel=True` |
| **不检查** spark_config | AWS_EMR 分支完全不读 spark_config |

### 入口 B2：ComputeService.build()

**代码位置**：`mage_ai/services/compute/models.py` L147-L156

**调用方**：Mage 前端集群管理页面

```python
@classmethod
def build(self, project, with_clusters=False):
    service_class = self
    if project and project.spark_config:      # 第 1 关
        if project.emr_config:                 # 第 2 关
            service_class = AWSEMRComputeService
    return service_class(project=project, with_clusters=with_clusters)
```

**入口 B2 的判断条件**：

| 条件 | 说明 |
|------|------|
| `project.spark_config` 非 None | 经 `or None` 转换后，空 dict `{}` 也会变成 `None` |
| `project.emr_config` 非 None | 经 `or None` 转换后，空 dict `{}` 也会变成 `None` |
| **不检查** kernel | 此路径不关心 kernel 类型 |

### 配置值的完整传递链路

#### emr_config 的传递链

```
metadata.yaml                    repo_config                    project
─────────────                   ───────────                    ───────
不写 emr_config       →  {}.get('emr_config') or dict()
                        = {}               →        {}.emr_config or None = None

emr_config: {}        →  {}.get('emr_config') or dict()
                        = {}               →        {}.emr_config or None = None

emr_config:           →  {}.get('emr_config') or dict()
  master_instance_type: r5.xlarge  = {master...: r5...}  →  {master...: r5...}
```

**结论**：`project.emr_config` 永远不会是 `{}`，要么 `None` 要么非空 dict。

#### spark_config 的传递链

```
metadata.yaml                    repo_config                    project
─────────────                   ───────────                    ───────
不写 spark_config      →  {}.get('spark_config')
                        = None             →       None or None = None

spark_config: {}       →  {}.get('spark_config')
                        = {}               →       {} or None = None

spark_config:          →  {}.get('spark_config')
  app_name: my-app       = {app_name: my-app}  →  {app_name: my-app}
```

**结论**：`project.spark_config` 也永远不会是 `{}`，要么 `None` 要么非空 dict。

### 入口 B1 和 B2 的完整真值表

| # | metadata.yaml 中 | repo_config.emr_config | repo_config.spark_config | project.emr_config | project.spark_config | B1 (utils.py) AWS_EMR? | B2 (compute/models.py) AWSEMR? |
|---|-------------------|------------------------|--------------------------|--------------------|----------------------|------------------------|--------------------------------|
| 1 | 都不写 | `{}` | `None` | `None` | `None` | ❌ `if {}` → False | ❌ `if None` → False |
| 2 | `emr_config: {}` | `{}` | `None` | `None` | `None` | ❌ `if {}` → False | ❌ `if None` → False |
| 3 | `emr_config: {}` + `spark_config: {}` | `{}` | `{}` | `None` | `None` | ❌ `if {}` → False | ❌ `if None` → False |
| 4 | `emr_config:` 有子项 | `{有key}` | `None` | `{有key}` | `None` | ✅ (需 kernel=PYSPARK) | ❌ `if None` → False |
| 5 | `emr_config:` 有子项 + `spark_config: {}` | `{有key}` | `{}` | `{有key}` | `None` | ✅ (需 kernel=PYSPARK) | ❌ `{} or None = None` → False |
| 6 | `emr_config:` 有子项 + `spark_config:` 有子项 | `{有key}` | `{有key}` | `{有key}` | `{有key}` | ✅ (需 kernel=PYSPARK) | ✅ |

### 两个入口的差异总结

| 维度 | B1 `get_compute_service()` | B2 `ComputeService.build()` |
|------|----------------------------|------------------------------|
| 数据源 | `repo_config` 直接读取 | `project` 属性（经 `or None` 转换） |
| emr_config | 必须非空 dict | 必须非空 dict（`{}` 被 `or None` 过滤为 `None`） |
| spark_config | **不检查** | 必须非空 dict（`{}` 被 `or None` 过滤为 `None`） |
| kernel 条件 | kernel=PYSPARK 或 ignore_active_kernel | **不检查** |
| 用途 | 决定 API.build() 走 AwsEmrAPI 还是 LocalAPI | 决定集群管理页显示哪种 ComputeService |

### 路径 B 的完整前置条件

要使 Spark UI 监控路由走 EMR 通道：

1. ✅ `metadata.yaml` 中 `emr_config` 段至少配置一个子项（如 `master_instance_type`）
2. ✅ `metadata.yaml` 中 `spark_config` 段至少配置一个子项（如 `app_name`），**空 dict `{}` 等效于缺失**
3. ✅ 当前 kernel 是 `pyspark`，或调用方传 `ignore_active_kernel=True`（仅 B1 需要）

---

## 四、两条路径的对照总结

### 独立生效示例

| 场景 | 路径 A（远程执行） | 路径 B（监控路由） |
|------|-------------------|-------------------|
| Pipeline type=pyspark + emr_config 有子项 + remote_variables_dir 有 + spark_config 未配 | ✅ 正常提交 EMR Step | ❌ B2 不通，Spark UI 空白 |
| Pipeline type=pyspark + emr_config 有子项 + remote_variables_dir 有 + spark_config 有子项 | ✅ 正常提交 EMR Step | ✅ Spark UI 通过 SSH 隧道访问 |
| Pipeline type=python + DEFAULT_EXECUTOR_TYPE=pyspark + emr_config 空 | ❌ EmrResourceManager 报错（缺 s3_bucket） | ❌ 两个入口都不通 |
| 非 PYSPARK Pipeline + emr_config 有子项 + spark_config 有子项 | ❌ 走本地 BlockExecutor | ✅ B1/B2 都走 EMR 路由（但无实际集群可连） |

### 配置依赖矩阵

| 配置项 | 路径 A 是否依赖 | 路径 B 是否依赖 | 说明 |
|--------|:---:|:---:|------|
| `pipeline.type == pyspark` | ✅ | ❌ | 执行器选择的关键条件 |
| `executor_type: pyspark` | ✅ | ❌ | Pipeline/Block 级配置 |
| `DEFAULT_EXECUTOR_TYPE` | ✅ | ❌ | 环境变量兜底 |
| `remote_variables_dir: s3://...` | ✅ | ❌ | 缺了直接报错 |
| `emr_config` 有实际子项 | 间接（EmrConfig 默认值兜底） | ✅ | B1 和 B2 都检查 |
| `spark_config` 有实际子项 | ❌ | ✅（仅 B2） | B1 不检查 |
| kernel == PYSPARK | ❌ | ✅（仅 B1） | 执行器不检查 |

> ⚠️ **emr_config 在路径 A 中的角色**：PySparkPipelineExecutor 不检查 emr_config 是否为空，空 dict 会传给 `EmrConfig.load()`，后者用硬编码默认值（如 `r5.4xlarge`、1 个 slave 节点）填充。所以 emr_config 对路径 A 是「有更好，没有也能跑（用默认值）」，而对路径 B 是「必须有（否则路由不通）」。

---

## 五、配置的完整加载链路

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

## 六、结果落点全图：状态与监控信息的精确边界

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

## 七、远程执行的完整数据流图（按代码事实校正版）

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
│  3. ComputeService.build(project)    ← 入口B                         │
│     ├─ project.spark_config 非空 dict（{} 被 or None 转为 None！）   │
│     └─ project.emr_config 非空 dict（{} 被 or None 转为 None！）     │
│         → AWSEMRComputeService                                      │
│                                                                      │
│  4. get_compute_service()            ← 入口A                         │
│     ├─ repo_config.emr_config 非空 dict（不检查 spark_config）       │
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

## 八、关键文件索引（相对路径）

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

## 九、与代码事实核对的关键结论

### 结论 1：远程执行和监控路由是两条独立的代码路径

| 维度 | 路径 A（远程执行） | 路径 B（监控路由） |
|------|-------------------|-------------------|
| 入口 | ExecutorFactory → PySparkExecutor | get_compute_service() / ComputeService.build() |
| 关键配置 | executor_type + remote_variables_dir | emr_config + spark_config + kernel |
| 检查 spark_config？ | ❌ | ✅（B2 要求） |
| 检查 kernel？ | ❌ | ✅（B1 要求） |

两条路径可以独立生效。Pipeline 在 EMR 上跑起来了（路径 A 通），不代表 Spark UI 能看到（路径 B 可能不通）。

### 结论 2：路径 A 的唯一硬约束是 s3_bucket

PySpark 执行器初始化时，不检查 emr_config 是否为空、不检查 spark_config、不检查 kernel。唯一会导致初始化失败的是 `s3_bucket is None`，即 metadata.yaml 中缺少 `remote_variables_dir: s3://bucket/path`。

emr_config 为空 dict `{}` 时，`EmrConfig.load()` 用硬编码默认值填充，集群照样能创建。

### 结论 3：路径 B 中空 dict `{}` 等效于缺失

`project/__init__.py` L155 和 L159 的 `or None` 转换，把空 dict `{}` 统一转为 `None`：

| metadata.yaml 写法 | repo_config 层 | project 层 | `if` 判断结果 |
|---------------------|----------------|------------|---------------|
| 不写 | `None`（spark_config）或 `{}`（emr_config） | `None` | False |
| 写 `{}` | `{}` | `None`（`{} or None`） | False |
| 写有子项 | `{有key}` | `{有key}` | True |

`spark_config: {}` 和完全不写 spark_config **效果完全相同**。

### 结论 4：路径 B 的两个入口要求不同

| 入口 | 检查 spark_config | 检查 kernel | 用途 |
|------|-------------------|-------------|------|
| B1 `get_compute_service()` | ❌ | ✅（PYSPARK 或 ignore_active_kernel） | API.build() 路由 |
| B2 `ComputeService.build()` | ✅（非空 dict） | ❌ | 集群管理页 |

要 Spark UI 完整走 EMR 通道，两个入口都需要通过，所以 `spark_config` 必须**至少写一个实际的子项**。

### 结论 5：update_status 参数在 PySpark 执行器中被完全忽略

- `PySparkPipelineExecutor.execute()` 有 `update_status` 参数，但方法体内**完全不用**
- `spark_script.jinja` 中的 `update_status=False` 是**硬编码**，不是传参
- 两层都不写 Mage DB，状态更新完全依赖上层调用方

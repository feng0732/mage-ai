# Spark / EMR 集成流程：入口选择条件与结果落点

> 本文档聚焦两个核心问题：
> 1. **什么配置会触发远程 EMR 执行**（而非本地 Python 执行）
> 2. **状态与监控信息落到哪里**（每类数据的写入位置、读取方式、是否可观测）

---

## 一、触发远程执行的三层判定链

Mage 决定「是否走 EMR 远程执行」由三层判定叠加而成，任何一层不满足都不会触发 EMR。

### 第 1 层：Pipeline 类型 → ExecutorType

**代码位置**：`mage_ai/data_preparation/executors/executor_factory.py` L24-L37

```
ExecutorFactory.get_pipeline_executor_type(pipeline)
  │
  ├─ pipeline.type == PipelineType.PYSPARK ?
  │     → YES：强制 executor_type = PYSPARK
  │
  └─ 否则：executor_type = pipeline.get_executor_type()
        │
        ├─ 返回值 == LOCAL_PYTHON 或 None ?
        │     → 回退到 env[DEFAULT_EXECUTOR_TYPE]，默认 LOCAL_PYTHON
        │
        └─ 返回值为其他（如 "pyspark"）?
              → 使用该值
```

**触发条件总结**（Pipeline 维度，满足任一即触发）：

| 条件 | 配置位置 | 配置写法 |
|------|----------|----------|
| **A. Pipeline type = pyspark** | Pipeline 的 `metadata.yaml` | `type: pyspark` |
| **B. Pipeline executor_type = pyspark** | Pipeline 的 `metadata.yaml` | `executor_type: pyspark` |
| **C. 环境变量 DEFAULT_EXECUTOR_TYPE = pyspark** | Mage 服务启动环境 | `export DEFAULT_EXECUTOR_TYPE=pyspark` |

**Pipeline metadata.yaml 示例**（`mage_ai/data_preparation/templates/pipeline/metadata.yaml`）：
```yaml
name: my_pipeline
uuid: abc123
type: pyspark          # ← 条件 A：直接声明 pyspark 类型
executor_type: pyspark # ← 条件 B：声明执行器类型
blocks:
  ...
```

### 第 2 层：Block 级选择逻辑

**代码位置**：`mage_ai/data_preparation/executors/executor_factory.py` L127-L144

```
ExecutorFactory.get_block_executor(pipeline, block_uuid)
  │
  ├─ pipeline.type == PYSPARK && (block.type != SENSOR || is_pyspark_code(block.content)) ?
  │     → YES：executor_type = PYSPARK
  │
  └─ 否则：executor_type = block.get_executor_type()
        │
        ├─ LOCAL_PYTHON 或 None → 回退到 DEFAULT_EXECUTOR_TYPE
        └─ 其他值 → 使用该值
```

**关键函数** `is_pyspark_code()`（`mage_ai/shared/code.py` L1-L2）：
```python
def is_pyspark_code(code: str):
    return '\'spark\'' in code or '"spark"' in code
```
只要 Block 代码中出现字符串 `'spark'` 或 `"spark"`，且 Pipeline 类型是 PYSPARK，就会被选为 PySpark 执行器。

**Block 级触发条件**（满足任一）：

| 条件 | 说明 |
|------|------|
| Pipeline type=pyspark + Block 不是 SENSOR + Block 代码含 `spark` 字面量 | 自动推断 |
| Block 自身 `executor_type: pyspark` | Block metadata 显式配置 |
| Block 为 LOCAL_PYTHON 但 DEFAULT_EXECUTOR_TYPE=pyspark | 环境变量兜底 |

### 第 3 层：计算服务类型判定（监控 / 交互层）

**代码位置**：`mage_ai/services/spark/utils.py` L9-L37

```
get_compute_service(repo_config)
  │
  ├─ repo_config.emr_config 存在 && (kernel==pyspark || ignore_active_kernel) ?
  │     → ComputeServiceUUID.AWS_EMR
  │
  ├─ is_spark_env() && spark_config.spark_master == 'local' ?
  │     → ComputeServiceUUID.STANDALONE_CLUSTER
  │
  └─ 都不满足 → None
```

以及 `ComputeService.build()`（`mage_ai/services/compute/models.py` L147-L156）：
```
ComputeService.build(project)
  │
  ├─ project.spark_config 存在 && project.emr_config 存在 ?
  │     → AWSEMRComputeService
  │
  └─ 否则 → StandaloneClusterComputeService
```

### 三层判定的关系图

```
                    ┌─────────────────────────┐
                    │   第 1 层：执行器选择      │
                    │   Pipeline/Block 配置     │
                    │   → 决定用哪个 Executor    │
                    └────────────┬────────────┘
                                 │ executor_type == PYSPARK
                                 ▼
                    ┌─────────────────────────┐
                    │   第 2 层：配置合并        │
                    │   repo_config.emr_config  │
                    │   + pipeline.executor_config
                    │   + block.executor_config │
                    │   → EmrConfig 实例        │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   第 3 层：服务类型路由    │
                    │   emr_config 存在 ?       │
                    │   → AWS_EMR vs LOCAL     │
                    │   → AwsEmrAPI vs LocalAPI│
                    └─────────────────────────┘
```

---

## 二、配置的完整加载链路

### 项目级配置（metadata.yaml）→ RepoConfig

**代码位置**：`mage_ai/data_preparation/repo_manager.py` L94-L164

```yaml
# 项目根目录/metadata.yaml
emr_config:                          # ← 读取为 repo_config.emr_config
  master_instance_type: 'r5.4xlarge'
  slave_instance_type: 'r5.4xlarge'
  master_security_group: 'sg-xxx'
  slave_security_group: 'sg-yyy'
  ec2_key_name: 'my-key-pair'
  ec2_key_path: '/path/to/key.pem'   # ← SSH 隧道需要
  bootstrap_script_path: null         # ← 自定义 bootstrap 覆盖默认
  spark_jars: []                      # ← 远程 Step 的 --jars 参数
  scaling_policy: null                # ← 托管扩缩容
  master_spark_properties: {}         # ← EMR Master spark-defaults
  slave_spark_properties: {}          # ← EMR Core spark-defaults

spark_config:                         # ← 读取为 repo_config.spark_config
  app_name: 'my spark app'            #    用于本地 SparkSession
  spark_master: 'local'               #    local / yarn / spark://host:port
  spark_jars: []
  executor_env: {}
  others: {}

remote_variables_dir: s3://bucket/path_prefix   # ← 解析出 s3_bucket + s3_path_prefix
```

加载链：
```
metadata.yaml → load_yaml → repo_config
                                    ├─ .emr_config = repo_config.get('emr_config') or dict()
                                    ├─ .spark_config = repo_config.get('spark_config')
                                    ├─ .s3_bucket = 解析 remote_variables_dir
                                    └─ .s3_path_prefix = 解析 remote_variables_dir
```

### Pipeline 级配置 → 合并到 EmrConfig

**代码位置**：`mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` L18-L23

```python
self.emr_config = self.pipeline.repo_config.emr_config or dict()  # 项目级兜底
if self.pipeline.executor_config is not None:
    self.emr_config = merge_dict(self.emr_config, self.pipeline.executor_config)  # Pipeline 级覆盖
self.emr_config = EmrConfig.load(config=self.emr_config)
```

### Block 级配置 → 再覆盖一层

**代码位置**：`mage_ai/data_preparation/executors/pyspark_block_executor.py` L31-L36

```python
self.executor_config = self.pipeline.repo_config.emr_config          # 项目级
if self.pipeline.executor_config is not None:
    self.executor_config = merge_dict(self.executor_config, self.pipeline.executor_config)  # Pipeline 级
if self.block.executor_config is not None:
    self.executor_config = merge_dict(self.executor_config, self.block.executor_config)     # Block 级
self.executor_config = EmrConfig.load(config=self.executor_config)
```

### 配置优先级（高 → 低）

```
Block.executor_config     ← 最细粒度，可覆盖单个 Block 的集群参数
  ↓ merge_dict
Pipeline.executor_config  ← Pipeline 维度，可覆盖特定 Pipeline 的集群参数
  ↓ merge_dict
repo_config.emr_config    ← 项目全局默认，metadata.yaml 中的 emr_config 段
```

---

## 三、结果落点全图：状态与监控信息去哪了

### 3.1 执行状态落点

| 状态类型 | 写入位置 | 写入方 | 读取方 | 时效性 |
|----------|----------|--------|--------|--------|
| **EMR Step 状态** | AWS EMR API（实时） | EMR 服务 | `emr.submit_spark_job()` 中的 `__status_poller` 轮询 | 实时，轮询间隔 30s+ |
| **EMR Step 状态** | Mage 控制台 stdout | `__status_poller` print 输出 | 人工查看日志 | 仅运行时 |
| **EMR 集群日志** | `s3://{bucket}/{prefix}/logs/` | EMR 自动写入 | S3 浏览 / Athena 查询 | 延迟 ~5min |
| **Spark 异常栈** | `s3://{bucket}/{prefix}/logs/` | spark_script.jinja 中 ErrorLogging | S3 浏览 | 仅异常时 |
| **Pipeline/Block 运行状态** | Mage 内部 DB (runs 表) | ❌ **远程执行不写入**（`update_status=False`） | Mage UI | **不更新** |

**关键细节**：远程 EMR 执行时，spark_script.jinja 中调用 `pipeline.execute(update_status=False)` 和 `block.execute_sync(update_status=False)`，这意味着：

- Mage UI 的 Pipeline Run 页面**不会**显示远程执行进度
- 只有 EMR Step 级别能看到 COMPLETED / FAILED
- Block 级别的输入输出变量**不会**被采集回 Mage

### 3.2 监控信息落点

| 监控数据 | 数据源 | 传输路径 | 展示位置 |
|----------|--------|----------|----------|
| **Spark Application 列表** | Spark UI REST API `/api/v1/applications` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 |
| **Spark Job 详情** | `/api/v1/applications/{id}/jobs` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 |
| **Spark Stage/Task 详情** | `/api/v1/applications/{id}/stages` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 |
| **Spark SQL 执行** | `/api/v1/applications/{id}/sql` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 |
| **Spark Executor 信息** | `/api/v1/applications/{id}/allexecutors` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 |
| **Spark 环境变量** | `/api/v1/applications/{id}/environment` | SSH 隧道 → AwsEmrAPI | Mage UI Spark 页面 |
| **Application 缓存** | `{variables_dir}/.spark/applications.json` | BaseSparkModel 本地文件 | Spark UI 不可达时的兜底展示 |
| **EMR 集群状态** | AWS EMR API `describe_cluster` | EmrClusterManager | Mage UI 集群管理页 |

### 3.3 SSH 隧道：EMR 监控的核心通道

**代码位置**：`mage_ai/services/ssh/aws/emr/models.py` L19-L188

```
Mage Server (本地)
  │
  │  SSH -i {ec2_key_path}
  │      -L {local_host}:{local_port}:{remote_host}:{remote_port}
  │      hadoop@{master_public_dns_name}
  │
  ▼
EMR Master Node
  │
  ├─ Livy Server  :8998   (交互式 Spark Session)
  ├─ Spark History :18080 (SPARK_UI_PORT_AWS_EMR)
  └─ Spark UI     :4040   (运行中 Application)
```

**隧道连通的前提条件**（缺一不可）：

1. `emr_config.ec2_key_path` 已配置（PEM 文件路径）
2. `emr_config.ec2_key_name` 已配置（EC2 Key Pair 名称）
3. EMR 集群 Master 的 22 端口可从 Mage Server 访问
4. EMR 集群处于 RUNNING / WAITING 状态
5. `SSHTunnel` 单例已初始化（`ec2_key_path` + `master_public_dns_name` 均不为空）

**隧道健康检查**（`models.py` L158-L173）：
```python
def precheck_access(self):
    test_socket = socket.socket(AF_INET, SOCK_STREAM)
    test_socket.settimeout(5)
    test_socket.connect((self.master_public_dns_name, SSH_PORT))  # 22 端口
    return True
```

### 3.4 数据输出落点

| 输出类型 | 落点 | 写入方 | 持久性 |
|----------|------|--------|--------|
| **Spark 表** | `saveAsTable('{db}.{table}')` → HDFS | io/spark.py `export()` | ❌ 集群销毁即丢失 |
| **Spark 表** (Glue) | Glue Catalog + S3 | io/spark.py `export()` | ✅ 持久 |
| **S3 文件** | `s3://user-defined-bucket/...` | 用户 Block 代码 | ✅ 持久 |
| **执行脚本** | `s3://{bucket}/{prefix}/scripts/` | EmrResourceManager | ✅ 持久（可覆盖） |
| **Bootstrap 脚本** | `s3://{bucket}/{prefix}/scripts/emr_bootstrap.sh` | EmrResourceManager | ✅ 持久（可覆盖） |
| **EMR 日志** | `s3://{bucket}/{prefix}/logs/` | EMR 服务自动 | ✅ 持久 |
| **Spark 错误栈** | `s3://{bucket}/{prefix}/logs/` | ErrorLogging (text 格式) | ✅ 持久 |

---

## 四、远程执行的完整数据流图

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Mage Server                                    │
│                                                                      │
│  1. ExecutorFactory 判定 executor_type == PYSPARK                    │
│     ↓                                                                │
│  2. PySparkPipelineExecutor / PySparkBlockExecutor                   │
│     ├─ 2a. 模板渲染 spark_script.jinja → execution_script            │
│     ├─ 2b. EmrResourceManager.upload_bootstrap_script() → S3        │
│     └─ 2c. s3.Client.upload(spark_script) → S3                      │
│                                                                      │
│  3. emr.submit_spark_job()                                           │
│     ├─ 3a. list_clusters(RUNNING/WAITING) → 找可用集群               │
│     ├─ 3b. 有可用集群 → add_job_flow_steps()                         │
│     │   无可用集群 → create_a_new_cluster() → run_job_flow()         │
│     └─ 3c. __status_poller() 轮询 Step 状态                          │
│              ↓ 每次轮询: describe_step → print(status)                │
│              ↓ 终态: COMPLETED / CANCELLED / FAILED / INTERRUPTED    │
│                                                                      │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│                                                                      │
│  监控通道 (独立于执行通道):                                            │
│                                                                      │
│  4. ComputeService.build(project)                                    │
│     ├─ emr_config + spark_config 存在 → AWSEMRComputeService        │
│     └─ 否则 → StandaloneClusterComputeService                        │
│                                                                      │
│  5. API.build()                                                      │
│     ├─ AWS_EMR → AwsEmrAPI                                           │
│     │   ├─ SSHTunnel 连接 (需要 ec2_key_path + master_dns)            │
│     │   └─ GET http://{tunnel_host}:{port}/api/v1/...                │
│     └─ STANDALONE → LocalAPI                                         │
│         └─ GET http://localhost:4040/api/v1/...                      │
│                                                                      │
│  6. Spark UI 数据 → 渲染到 Mage 前端                                  │
│     同时缓存到 {variables_dir}/.spark/applications.json               │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                       AWS EMR 集群                                   │
│                                                                      │
│  spark-submit --deploy-mode cluster s3://.../scripts/{uuid}.py       │
│     │                                                                │
│     ├─ SparkSession.builder.appName('My PyPi').getOrCreate()         │
│     │   ├─ Arrow 优化: spark.sql.execution.arrow.enabled=true       │
│     │   ├─ AQE: spark.sql.adaptive.enabled=true                     │
│     │   └─ SkewJoin: spark.sql.adaptive.skewJoin.enabled=true       │
│     │                                                                │
│     ├─ Pipeline(config=渲染后的配置, repo_config=远程配置)             │
│     │   └─ global_vars['spark'] = spark (注入 SparkSession)         │
│     │                                                                │
│     ├─ 执行: pipeline.execute(update_status=False) ← 不回写状态      │
│     │       或 block.execute_sync(update_status=False)               │
│     │                                                                │
│     └─ 异常: ErrorLogging                                            │
│         └─ error + traceback → S3 text 文件                          │
│              df.write.format('text').save({{ spark_log_path }})      │
│                                                                      │
│  日志: s3://{bucket}/{prefix}/logs/ (EMR 自动)                       │
│  Spark UI: :18080 (History) / :4040 (运行中)                         │
│  Livy: :8998 (交互式 Session)                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键文件索引（相对路径）

| 层级 | 文件路径 | 关键职责 |
|------|----------|----------|
| **执行器工厂** | `mage_ai/data_preparation/executors/executor_factory.py` | 判定 executor_type → 选择执行器 |
| **Pipeline 执行器** | `mage_ai/data_preparation/executors/pyspark_pipeline_executor.py` | Pipeline 级远程 EMR 执行 |
| **Block 执行器** | `mage_ai/data_preparation/executors/pyspark_block_executor.py` | Block 级远程 EMR 执行 |
| **EMR 核心服务** | `mage_ai/services/aws/emr/emr.py` | 集群创建、任务提交、步骤轮询 |
| **EMR 基础封装** | `mage_ai/services/aws/emr/emr_basics.py` | boto3 底层调用 |
| **EMR 配置** | `mage_ai/services/aws/emr/config.py` | EmrConfig 数据结构 |
| **EMR 启动器** | `mage_ai/services/aws/emr/launcher.py` | 独立集群创建入口 |
| **EMR 资源管理** | `mage_ai/services/aws/emr/resource_manager.py` | S3 路径管理 + bootstrap 上传 |
| **集群管理器** | `mage_ai/cluster_manager/aws/emr_cluster_manager.py` | 集群列表/创建/激活 sparkmagic |
| **配置加载** | `mage_ai/data_preparation/repo_manager.py` | metadata.yaml → RepoConfig |
| **Spark 会话** | `mage_ai/services/spark/spark.py` | SparkSession 获取（本地模式） |
| **Spark 配置** | `mage_ai/services/spark/config.py` | SparkConfig 数据结构 |
| **服务路由** | `mage_ai/services/spark/utils.py` | 计算服务类型判断 |
| **API 路由** | `mage_ai/services/sp/api/service.py` | LocalAPI / AwsEmrAPI 工厂 |
| **EMR API** | `mage_ai/services/spark/api/aws_emr.py` | EMR 模式 Spark UI 查询 |
| **本地 API** | `mage_ai/services/spark/api/local.py` | 本地模式 Spark UI 查询 |
| **API 基类** | `mage_ai/services/spark/api/base.py` | REST 请求抽象 |
| **API 常量** | `mage_ai/services/spark/api/constants.py` | SPARK_UI_HOST/PORT 定义 |
| **Spark 常量** | `mage_ai/services/spark/constants.py` | ComputeServiceUUID 枚举 |
| **Spark 模型基类** | `mage_ai/services/spark/models/base.py` | 缓存目录路径 |
| **Application 模型** | `mage_ai/services/spark/models/applications.py` | 缓存 + EMR ID 格式转换 |
| **Job 模型** | `mage_ai/services/spark/models/jobs.py` | JobStatus 枚举 |
| **SSH 隧道** | `mage_ai/services/ssh/aws/emr/models.py` | SSHTunnel 单例 |
| **计算服务** | `mage_ai/services/compute/models.py` | ComputeService.build() 路由 |
| **执行脚本模板** | `mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja` | 远程执行脚本 |
| **Bootstrap 模板** | `mage_ai/data_preparation/templates/pipeline_execution/emr_bootstrap.sh` | 集群节点初始化 |
| **Pipeline 元数据模板** | `mage_ai/data_preparation/templates/pipeline/metadata.yaml` | Pipeline 配置模板 |
| **项目元数据模板** | `mage_ai/data_preparation/templates/repo/metadata.yaml` | 项目 metadata.yaml 模板 |
| **IO 连接器** | `mage_ai/io/spark.py` | Spark 数据读写 |
| **API Mixin** | `mage_ai/api/resources/mixins/spark.py` | Spark 监控 REST API |
| **PySpark 代码检测** | `mage_ai/shared/code.py` | is_pyspark_code() |
| **枚举常量** | `mage_ai/data_preparation/models/constants.py` | ExecutorType / PipelineType |

---

## 六、容易遗漏的配置陷阱

### 陷阱 1：metadata.yaml 中 emr_config 为空字典 ≠ 未配置

`repo_manager.py` L135：
```python
self.emr_config = repo_config.get('emr_config') or dict()
```
即使 metadata.yaml 没有写 `emr_config`，`repo_config.emr_config` 也默认为 `{}`（非 None）。
这意味着 `get_compute_service()` 中的 `repo_config.emr_config` 判断**永远为 True**。

**但** `ComputeService.build()` 同时要求 `project.spark_config` 存在：
```python
if project.emr_config:    # {} 评估为 False
```
空 dict 在 Python 中为 falsy，所以 `emr_config: {}` 等价于未配置。

**结论**：必须在 metadata.yaml 中显式填入 `emr_config` 的至少一个子项，才能触发 AWS_EMR 路由。

### 陷阱 2：remote_variables_dir 是 S3 Bucket 的唯一来源

`repo_manager.py` L158-L164：
```python
if self.remote_variables_dir is not None and self.remote_variables_dir.startswith('s3://'):
    path_parts = self.remote_variables_dir.replace('s3://', '').split('/')
    self.s3_bucket = path_parts.pop(0)
    self.s3_path_prefix = '/'.join(path_parts)
```

如果不配置 `remote_variables_dir: s3://bucket/path`，则 `s3_bucket = None`，
`EmrResourceManager.__init__()` 会抛出异常：
```
Please specify the correct s3_bucket to initialize EMR cluster.
Add "remote_variables_dir: s3://[bucket]/[path]" to project's metadata.yaml file.
```

### 陷阱 3：Block executor_type 的隐藏覆盖

Block 的 `executor_type` 在 Pipeline `metadata.yaml` 的 block 配置中设置，默认为 `LOCAL_PYTHON`。
只有当 `pipeline.type == PYSPARK` 且 `is_pyspark_code(block.content)` 返回 True 时，
才会自动提升为 `PYSPARK`。

**注意**：如果 Block 是 SENSOR 类型，即使代码含 `spark` 也不会自动提升，
需要显式设置 `executor_type: pyspark`。

### 陷阱 4：SSH 隧道未建立时监控全黑

`AwsEmrAPI.ready_for_requests()` 返回 `SSHTunnel().is_active()`。
如果隧道未建立，所有 Spark UI 查询返回空字典 `{}`，前端显示空白。

隧道建立需要：
1. metadata.yaml 中配置 `ec2_key_name` 和 `ec2_key_path`
2. EMR 集群处于 RUNNING/WAITING
3. Mage Server 到 EMR Master 的 22 端口网络可达

### 陷阱 5：spark_script.jinja 中 update_status=False 的连锁影响

远程执行脚本中 `pipeline.execute(analyze_outputs=False, update_status=False)`，
意味着：
- ❌ Mage DB 不会记录 Block 级别的运行状态
- ❌ Block 的输入输出变量不会保存
- ❌ Pipeline Run 的完成时间不会更新
- ✅ EMR Step 级别的状态可通过 AWS Console / boto3 查询
- ✅ Spark 级别的 Job/Stage 可通过 Spark UI 查询（需 SSH 隧道）

**如果需要状态回写**，需要修改 spark_script.jinja 模板或添加回调机制。

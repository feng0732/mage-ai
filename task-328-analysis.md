# Spark / EMR 集成代码实现分析

> 分析日期：2026-06-16  
> 代码版本：328-mage-ai  
> 目的：从代码实现角度梳理 Spark/EMR 集成的 **入口**、**协作链路**、**结果落点**

---

## 一、整体架构概览

```
Mage UI / API
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│                   执行调度层 (Executors)                       │
│  PySparkPipelineExecutor  /  PySparkBlockExecutor             │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                   EMR 服务层 (services/aws/emr)               │
│  submit_spark_job → create_a_new_cluster / __add_step         │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
                 AWS EMR API (boto3)
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                    EMR 集群运行时                              │
│  Spark Master (Livy 8998)  ──  Spark UI (18080/20888)         │
│  Worker Nodes  ──  HDFS / S3                                 │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
                 结果输出 (S3 / Spark Tables)
```

---

## 二、集成入口（5 个关键入口）

### 入口 1：Pipeline 执行入口
**文件**：[pyspark_pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/executors/pyspark_pipeline_executor.py#L14-L85)

当 Pipeline 的 executor_config 配置为 `pyspark` 时，Mage 会使用 `PySparkPipelineExecutor` 作为执行器。

核心入口方法 `execute()`（[L32-L48](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/executors/pyspark_pipeline_executor.py#L32-L48)）：
```python
def execute(self, ...):
    self.upload_pipeline_execution_script(...)   # 1. 上传脚本到 S3
    self.resource_manager.upload_bootstrap_script()  # 2. 上传 bootstrap 脚本
    self.submit_spark_job()                  # 3. 提交 Spark Job 到 EMR
```

### 入口 2：单 Block 执行入口
**文件**：[pyspark_block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/executors/pyspark_block_executor.py#L16-L97)

当单个 Block 配置独立 executor 时使用。核心 `_execute()` 方法（[L38-L55](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/executors/pyspark_block_executor.py#L38-L55)）：

与 Pipeline 执行类似，但脚本路径不同：
- Pipeline：`s3://{bucket}/{prefix}/scripts/{pipeline_uuid}.py`
- Block：`s3://{bucket}/{prefix}/scripts/{pipeline_uuid}/{block_uuid}.py`

### 入口 3：集群管理入口
**文件**：[emr_cluster_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/cluster_manager/aws/emr_cluster_manager.py#L17-L98)

对外暴露 3 个核心方法：
| 方法 | 作用 | 关键行 |
|------|------|--------|
| `list_clusters()` | 列出 Mage 管理的 EMR 集群（通过 name=mage-data-prep 标签过滤） | [L21-L37](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/cluster_manager/aws/emr_cluster_manager.py#L21-L37) |
| `create_cluster()` | 创建新的 EMR 集群（空步骤，保活等待任务） | [L39-L49](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/cluster_manager/aws/emr_cluster_manager.py#L39-L49) |
| `set_active_cluster()` | 绑定活动集群 → 配置 sparkmagic → Livy 交互 | [L51-L95](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/cluster_manager/aws/emr_cluster_manager.py#L51-L95) |

### 入口 4：Spark UI 监控 API 入口
**文件**：[spark.py (mixins)](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/api/resources/mixins/spark.py#L9-L45)  
**服务路由**：[service.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/api/service.py#L7-L34)

`API.build()` 会根据计算服务类型自动路由：
- `STANDALONE_CLUSTER` → `LocalAPI`
- `AWS_EMR` → `AwsEmrAPI`（需要 SSH 隧道）

判断逻辑见 [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/utils.py#L9-L37) 的 `get_compute_service()`：
```
repo_config.emr_config 存在 + (pyspark kernel or ignore_active_kernel)
    → 返回 AWS_EMR
```

### 入口 5：Spark IO 数据访问入口
**文件**：[io/spark.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/io/spark.py#L10-L230)

`Spark` 类继承自 `BaseSQLDatabase`，提供：
- `load()`：Spark SQL → Pandas DataFrame（限制 10M 行）
- `export()`：Pandas DataFrame → Spark 表（append/replace/fail）
- `execute()`：执行 DDL / DML

---

## 三、协作链路（3 条核心调用链）

### 链路 A：远程 Batch 执行（PySparkPipelineExecutor → EMR Step）

```
PySparkPipelineExecutor.execute()
  │
  ├─ upload_pipeline_execution_script()
  │   └─ 模板渲染 spark_script.jinja
  │       └─ 上传到 s3://{bucket}/{prefix}/scripts/{uuid}.py
  │           调用: s3.Client.upload()
  │
  ├─ resource_manager.upload_bootstrap_script()
  │   └─ 上传 emr_bootstrap.sh 到 S3
  │       (安装 mage-ai + numpy/pandas/sklearn 等依赖)
  │
  └─ submit_spark_job()
      └─ emr.submit_spark_job()   [emr.py L153-L260]
          │
          ├─ 查找可用集群
          │   ├─ list_clusters(RUNNING/WAITING)
          │   ├─ 按 WAITING 优先排序
          │   ├─ 检查步骤数: active < 15, total < 200
          │   └─ valid_cluster_ids = [...]
          │
          ├─ 有可用集群?
          │   ├─ YES → __add_step()  [emr.py L263-L290]
          │   │     └─ add_job_flow_steps(Steps=[spark-submit ...])
          │   │         └─ __status_poller() 轮询到 COMPLETED
          │   │
          │   └─ NO → create_a_new_cluster()  [emr.py L35-L119]
          │         ├─ run_job_flow(
          │         │     ReleaseLabel='emr-6.9.0',
          │         │     Applications=[Hadoop, Hive, Spark],
          │         │     StepConcurrencyLevel=256,
          │         │     Instances=emr_config.get_instances_config(),
          │         │     BootstrapActions=[emr_bootstrap.sh],
          │         │     Steps=[...],
          │         │   )
          │         ├─ put_auto_termination_policy(idle_timeout)
          │         └─ __status_poller() → WAITING
          │
          └─ 异常: Step 状态 != COMPLETED → raise Exception
```

**关键配置限制**（[emr.py L16-L17](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr.py#L16-L17)）：
```python
MAX_STEPS_IN_CLUSTER = 255 - 55  # =200 (单集群最大步骤数)
MAX_RUNNING_OR_PENDING_STEPS = 15  # 并行活跃步骤上限
```

### 链路 B：EMR 上的脚本内部执行（运行在 Spark Driver 上）

脚本模板来源：[spark_script.jinja](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja#L1-L82)

```
spark-submit --deploy-mode cluster s3://.../scripts/{uuid}.py
  │
  └─ SparkSession.builder.appName('My PyPi').getOrCreate()
      │
      ├─ Arrow 优化:
      │   ├─ spark.sql.execution.arrow.enabled = true
      │   ├─ spark.sql.adaptive.enabled = true
      │   └─ spark.sql.adaptive.skewJoin.enabled = true
      │
      ├─ ErrorLogging(spark, {{ spark_log_path }})
      │   └─ 异常时:
      │       ├─ df = createDataFrame([error, traceback], StringType)
      │       └─ df.write.format('text').save({{ spark_log_path }})
      │           → s3://{bucket}/{prefix}/logs/
      │
      ├─ pipeline = Pipeline(uuid={{ uuid }}, config={{ config }}, ...)
      │
      ├─ global_vars['spark'] = spark  (注入 SparkSession)
      │
      └─ 分支:
          ├─ block_uuid is None → asyncio.run(pipeline.execute())
          │                      (执行整个 Pipeline)
          └─ block_uuid != None → block.execute_sync() + block.run_tests()
                                 (执行单个 Block)
```

### 链路 C：Spark UI 监控（SSH 隧道 + REST API）

```
Mage Web UI
  │
  ▼
HTTP GET /api/spark/applications  (mixins/spark.py)
  │
  ▼
API.build()  [service.py]
  │
  ├─ get_compute_service() == AWS_EMR ?
  │
  ├─ YES → AwsEmrAPI()  [aws_emr.py]
  │   │
  │   ├─ ready_for_requests()
  │   │   └─ SSHTunnel().is_active()  [models.py L35-L38]
  │   │
  │   ├─ spark_ui_url  [aws_emr.py L17-L33]
  │   │   └─ SSHTunnel.connection_details()
  │   │       → http://{host}:{port}
  │   │
  │   └─ applications_sync()
  │       └─ GET http://{tunnel_host}:{port}/api/v1/applications
  │
  └─ NO → LocalAPI()  [local.py]
      └─ GET http://localhost:4040/api/v1/applications
```

**SSH 隧道核心**：[models.py (SSHTunnel)](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/ssh/aws/emr/models.py#L19-L304)

```
SSHTunnel 单例模式:
  local_bind_address → 127.0.0.1:{auto_port}
  remote_bind_address → {master_dns}:18080 (SPARK_UI_PORT_AWS_EMR)
  ssh_pkey → emr_config.ec2_key_path
  ssh_username → hadoop (默认)
```

---

## 四、结果落点（5 类输出位置）

### 落点 1：S3 脚本 & 日志存储
由 [EmrResourceManager](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/resource_manager.py#L7-L43) 统一管理路径：

| 资源 | S3 路径 |
|------|---------|
| Bootstrap 脚本 | `s3://{bucket}/{prefix}/scripts/emr_bootstrap.sh` |
| Pipeline 执行脚本 | `s3://{bucket}/{prefix}/scripts/{pipeline_uuid}.py` |
| Block 执行脚本 | `s3://{bucket}/{prefix}/scripts/{pipeline_uuid}/{block_uuid}.py` |
| EMR 集群日志 | `s3://{bucket}/{prefix}/logs` |
| 执行错误日志 | `s3://{bucket}/{prefix}/logs` (由 spark_script.jinja ErrorLogging 写入) |

### 落点 2：Spark 内部表 (Hive Metastore / Glue)
通过 [io/spark.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/io/spark.py#L155-L230) 的 `export()` 写入：

```python
spark_df.write.mode('append').saveAsTable(f'{database}.{table_name}')
```

存储介质取决于 EMR 集群配置：
- 默认：HDFS（集群销毁则丢失）
- 若配置 Glue Metastore：持久化到 Glue Catalog + S3

### 落点 3：S3 Data Output（用户代码自定义）
用户 Block 代码中显式写出，例如：
```python
df.write.parquet('s3://my-bucket/output/table1/')
```

这部分不由 Mage 框架管理，完全取决于用户代码。

### 落点 4：本地 Application 缓存（Mage Server 端）
Spark 应用元数据缓存在 Mage 服务器本地：  
[applications.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/models/applications.py#L58-L109)

```
.spark/applications.json
  └─ { calculated_id: { id, name, attempts, spark_ui_url, ... } }
```

用途：当 Spark UI 不可达时，前端仍可展示历史 Application 信息。

### 落点 5：Pipeline/Block 的执行元数据
与普通执行器一致，写入 Mage 内部数据库（Runs 表 / Blocks 状态），  
由 spark_script.jinja 中 `pipeline.execute(update_status=False)` 控制——  
**注意**：远程执行时 `update_status=False`，状态同步需依赖其他机制。

---

## 五、关键配置数据结构

### EmrConfig（集群配置）
**文件**：[config.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/config.py#L26-L119)

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `master_instance_type` | `r5.4xlarge` | Master 机型 |
| `slave_instance_type` | `r5.4xlarge` | Core 机型 |
| `slave_instance_count` | `1` | Core 节点数 |
| `bootstrap_script_path` | `None` | 自定义 bootstrap 脚本（覆盖默认） |
| `ec2_key_name` | `None` | EC2 key pair 名称 |
| `master_security_group` | `None` | Master 安全组 ID |
| `slave_security_group` | `None` | Slave 安全组 ID |
| `scaling_policy` | `None` | 托管扩缩容策略 (min/max capacity) |
| `spark_jars` | `[]` | 附加 jar 包列表 |
| `master_spark_properties` | `{}` | Master 的 spark-defaults 覆盖 |
| `slave_spark_properties` | `{}` | Core 的 spark-defaults 覆盖 |

**实例 → 内存映射**：
```
r5.xlarge   → 8000M
r5.2xlarge  → 16000M
其他        → 32000M (默认)
```

### SparkConfig（本地会话配置）
**文件**：[spark/config.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/config.py#L8-L17)

用于 `SparkSession.builder` 的本地/独立集群配置：
- `app_name`, `spark_master`, `spark_home`
- `executor_env`, `spark_jars`, `others`
- `use_custom_session`：使用用户手动创建的 SparkSession

### Bootstrap 脚本默认内容
**文件**：[emr_bootstrap.sh](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/emr_bootstrap.sh#L1-L11)

安装依赖链：
```
yum install git-core, python3-devel
  → pip install mage-ai (from GitHub master)
  → numpy==1.19.2, scipy==1.2.2, pandas==1.1.3
  → pyarrow==0.13.0, scikit-learn==0.24.1
  → boto3==1.24.19
```

---

## 六、核心文件索引表

| 层级 | 文件 | 关键职责 |
|------|------|----------|
| **执行器** | [pyspark_pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/executors/pyspark_pipeline_executor.py) | Pipeline 级远程 EMR 执行 |
| **执行器** | [pyspark_block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/executors/pyspark_block_executor.py) | Block 级远程 EMR 执行 |
| **EMR 服务** | [emr.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr.py) | 核心：集群创建、任务提交、步骤轮询 |
| **EMR 服务** | [emr_basics.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr_basics.py) | 底层 boto3 调用封装 |
| **EMR 服务** | [config.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/config.py) | EmrConfig 数据结构 |
| **EMR 服务** | [launcher.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/launcher.py) | 独立集群创建入口 |
| **EMR 服务** | [resource_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/resource_manager.py) | S3 路径管理 + bootstrap 上传 |
| **集群管理** | [emr_cluster_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/cluster_manager/aws/emr_cluster_manager.py) | 集群列表/创建/激活（sparkmagic） |
| **Spark 服务** | [spark.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/spark.py) | SparkSession 获取（本地模式） |
| **Spark 服务** | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/utils.py) | 计算服务类型判断 |
| **Spark API** | [service.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/api/service.py) | LocalAPI / AwsEmrAPI 路由 |
| **Spark API** | [aws_emr.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/api/aws_emr.py) | EMR 模式 Spark UI 查询 |
| **Spark API** | [local.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/spark/api/local.py) | 本地模式 Spark UI 查询 |
| **SSH 隧道** | [models.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/ssh/aws/emr/models.py) | SSHTunnel 单例（端口转发） |
| **模板** | [spark_script.jinja](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja) | 远程执行 Python 脚本模板 |
| **模板** | [emr_bootstrap.sh](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/emr_bootstrap.sh) | 集群节点初始化脚本 |
| **IO** | [io/spark.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/io/spark.py) | Spark 数据读写连接器 |
| **API Mixin** | [mixins/spark.py](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/api/resources/mixins/spark.py) | Spark 监控 REST API 复用逻辑 |

---

## 七、容易看漏的关键细节（6 个坑点）

### 坑点 1：Step 并发数硬限制
`MAX_RUNNING_OR_PENDING_STEPS = 15`（[emr.py L17](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr.py#L17)）——  
超过就会创建新集群，导致成本飙升，而非排队。

### 坑点 2：update_status=False
spark_script.jinja 中远程执行时**不更新数据库状态**（[L66-L77](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja#L66-L77)），  
执行结果需要通过其他机制同步回 Mage 主服务。

### 坑点 3：StepConcurrencyLevel=256
集群创建时设置了 256 并发（[emr.py L67](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr.py#L67)），但实际被 L17 的 15 限制，  
两个参数存在不一致，仅作配置冗余。

### 坑点 4：错误日志只写 S3
ErrorLogging 类（[spark_script.jinja L15-L39](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/spark_script.jinja#L15-L39)）  
捕获异常后只写入 S3 text 文件，Mage 主服务端 UI 看不到详细错误栈，需要去 S3 查日志。

### 坑点 5：Bootstrap 从 GitHub 安装 mage-ai
默认 `pip install git+https://github.com/mage-ai/mage-ai.git`（[emr_bootstrap.sh L5](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/data_preparation/templates/pipeline_execution/emr_bootstrap.sh#L5)），  
这会装 master 分支而非当前运行版本，可能出现版本不兼容。  
生产环境必须自定义 `emr_config.bootstrap_script_path` 指定固定版本。

### 坑点 6：EMR 版本硬编码
`ReleaseLabel='emr-6.9.0'` 硬编码在两处（[emr.py L60](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr.py#L60) 和 [emr_basics.py L52](file:///d:/fz/0601/solo-dogfeeding/code/328-mage-ai/mage_ai/services/aws/emr/emr_basics.py#L52)），  
升级 EMR 版本需修改源码。

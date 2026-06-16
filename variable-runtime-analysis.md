# Mage AI 全局 Variable 到 Pipeline 执行链路 —— 代码全景图

## 一、变量体系三层数据模型

Mage AI 的变量体系由 **3 类存储** 和 **2 种接入方式** 构成：

```
┌────────────────────────────────────────────────────────────────────┐
│  ① 全局变量 (Global Variables)                                     │
│     存储: pipeline YAML 文件的 variables 字段                       │
│     位置: pipelines/<uuid>/pipeline.yaml → variables: {key: val}   │
│     读写: Pipeline.variables 属性 / Pipeline.update_global_variable│
│     特点: 与代码一起版本控制，序列化到 YAML                           │
├────────────────────────────────────────────────────────────────────┤
│  ② 中间变量 (Block Output Variables)                               │
│     存储: variables_dir/pipelines/<uuid>/variables/<block>/<var>   │
│     读写: VariableManager.add_variable / get_variable              │
│     特点: Block 执行后的输出数据，持久化到磁盘/S3/GCS                 │
├────────────────────────────────────────────────────────────────────┤
│  ③ 运行时变量 (Runtime Variables)                                  │
│     存储: DB PipelineRun.variables (JSON)                          │
│     来源: 触发时传入 / CLI --runtime-vars / 事件变量                 │
│     特点: 每次运行实例独有，不持久化到代码                            │
├────────────────────────────────────────────────────────────────────┤
│  模板变量 (Template Variables) — 特殊注入层                         │
│     函数: env_var(), mage_secret_var(), aws_secret_var()           │
│     场景: metadata.yaml / io_config.yaml 的 Jinja2 渲染            │
│     特点: 不是独立变量，是渲染时的函数注入                             │
└────────────────────────────────────────────────────────────────────┘
```

---

## 二、VariableManager CRUD 接口详解

[variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py) 是中间变量的统一管理入口。

### 2.1 工厂方法：存储后端选择

```python
VariableManager.get_manager(repo_path, variables_dir)
    ↓
variables_dir.startswith('s3://')  → S3VariableManager   (S3Storage)
variables_dir.startswith('gs://')  → GCSVariableManager  (GCSStorage)
其他                               → VariableManager     (LocalStorage)
```

### 2.2 写接口

| 方法 | 签名 | 调用方 | 说明 |
|------|------|--------|------|
| `add_variable` | `(pipeline_uuid, block_uuid, variable_uuid, data, ...)` | Block 执行完成后 | 类型推断 → Variable 写入 → 元数据写入 |
| `add_variable_async` | 同上（异步） | Block 异步执行 | 先 delete 再 write_data_async |
| `add_variable_types` | `(pipeline_uuid, block_uuid, variable_uuid, variable_types)` | ITERABLE 类型 | 仅写元数据，不写数据 |

**add_variable 写入流程**：

```
1. infer_variable_type(data) → 推断类型 (DATAFRAME / POLARS_DATAFRAME / JSON ...)
2. Variable(uuid, pipeline_path, block_uuid) → 构造 Variable 对象
3. write_batch_settings.mode != APPEND? → variable.delete() 先删旧数据
4. MEMORY_MANAGER_V2 + 基本可迭代? → 递归拆分子变量 (DataGenerator)
   └→ 每个子项 add_variable 递归，父变量 variable_type = ITERABLE
5. else → variable.write_data(data) → 按类型分发:
   ├─ DATAFRAME          → __write_parquet
   ├─ POLARS_DATAFRAME   → __write_polars_dataframe
   ├─ SPARK_DATAFRAME    → __write_spark_parquet
   ├─ GEO_DATAFRAME      → __write_geo_dataframe
   ├─ MATRIX_SPARSE      → __write_matrix_sparse
   ├─ SERIES_PANDAS      → __write_series_pandas → fallback __write_json
   ├─ DICTIONARY_COMPLEX → __save_complex_object → __write_json
   └─ 其他               → __should_save_object → __write_json
6. write_metadata() + __write_resource_usage()
```

### 2.3 读接口

| 方法 | 签名 | 返回值 | 说明 |
|------|------|--------|------|
| `get_variable` | `(pipeline_uuid, block_uuid, variable_uuid, ...)` | `Any` | 读数据值 |
| `get_variable_object` | `(pipeline_uuid, block_uuid, variable_uuid, ...)` | `Variable` | 读 Variable 对象（不读数据） |
| `get_variables_by_block` | `(pipeline_uuid, block_uuid, ...)` | `List[str]` | 列出 block 下所有变量 UUID |
| `get_variables_by_pipeline` | `(pipeline_uuid)` | `Dict[str, List[str]]` | 列出 pipeline 下所有 block 的变量 |

**get_variable 读取流程**：

```
1. get_variable_object() → 构造 Variable 对象
   ├─ variable_uuid 未指定? → get_first_data_output_variable_uuid() → 默认 'output_0'
   └─ spark + DATAFRAME? → variable_type = SPARK_DATAFRAME
2. variable.read_data() → 按类型分发:
   ├─ data_manager.readable()? → data_manager.read_sync (V2 内存管理)
   ├─ DATAFRAME / SERIES_PANDAS → __read_parquet + 列类型反序列化
   ├─ POLARS_DATAFRAME → __read_polars_parquet
   ├─ SPARK_DATAFRAME → __read_spark_parquet
   ├─ GEO_DATAFRAME → __read_geo_dataframe
   ├─ DATAFRAME_ANALYSIS → __read_dataframe_analysis
   └─ 其他 → __read_json (+ MATRIX_SPARSE/DICTIONARY_COMPLEX 后处理)
```

### 2.4 删接口

| 方法 | 签名 | 说明 |
|------|------|------|
| `delete_variable` | `(pipeline_uuid, block_uuid, variable_uuid)` | 删除变量数据 |
| `clean_variables` | `(pipeline_uuid=None)` | 按保留期批量清理 |

**delete_variable 流程**：

```
Variable(uuid, ...).delete()
├─ DATAFRAME          → __delete_parquet + __delete_json
├─ DATAFRAME_ANALYSIS → __delete_dataframe_analysis
└─ 其他               → __delete_json (删除目录/旧格式文件)
```

### 2.5 变量目录结构

```
$variables_dir/
  pipelines/
    <pipeline_uuid>/
      variables/
        <partition>/          ← 执行分区 (时间戳)
          <block_uuid>/
            <variable_uuid>/
              type.json       ← 元数据: {type: "dataframe", types: [...]}
              data.parquet    ← DataFrame 数据
              data.json       ← 非 DataFrame 数据
              sample_data.json / sample_data.parquet  ← 采样数据
              data_column_types.json  ← 列类型信息
              resource_usage.json     ← 资源使用统计
            global/           ← 全局变量在 VariableManager 中的存储
              <key>/
                data.json
```

---

## 三、Variable 底层模型

[variable.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/variable.py) 是单变量的读写核心。

### 3.1 变量类型体系

[constants.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/variables/constants.py) 定义：

| VariableType | 存储格式 | 说明 |
|-------------|---------|------|
| `DATAFRAME` | `data.parquet` + `sample_data.parquet` | Pandas DataFrame |
| `POLARS_DATAFRAME` | `data.parquet` | Polars DataFrame |
| `SPARK_DATAFRAME` | Parquet 目录 | Spark DataFrame |
| `GEO_DATAFRAME` | `data.sh` | GeoPandas |
| `SERIES_PANDAS` | `data.parquet` | Pandas Series |
| `SERIES_POLARS` | Parquet | Polars Series |
| `ITERABLE` | 子目录 0/, 1/, 2/... | 列表拆分（V2 内存管理） |
| `DICTIONARY_COMPLEX` | `data.json` + `data_column_types.json` | 复杂字典 |
| `LIST_COMPLEX` | `data.json` + `data_column_types.json` | 复杂列表 |
| `MATRIX_SPARSE` | `data.json` | SciPy 稀疏矩阵 |
| `MODEL_SKLEARN` | `model.joblib` | Sklearn 模型 |
| `MODEL_XGBOOST` | `model.ubj` | XGBoost 模型 |
| `CUSTOM_OBJECT` | `object.joblib` | 自定义对象 |
| `DATAFRAME_ANALYSIS` | `statistics.json` 等 | DataFrame 分析结果 |

### 3.2 类型推断链

[variable.py check_variable_type()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/variable.py#L208-L268)：

```
1. metadata_path 存在? → 读取 type.json → VariableType(metadata['type'])
2. V2 + part_uuids 存在? → 读取子目录 metadata → 类型列表 → ITERABLE
3. data.parquet 存在? → DATAFRAME
4. data.sh 存在? → GEO_DATAFRAME
5. *.parquet 存在 + spark? → SPARK_DATAFRAME
```

---

## 四、全局变量读写链路

### 4.1 读取：get_global_variables()

[variable_manager.py get_global_variables()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L504-L541)

```python
def get_global_variables(pipeline_uuid, pipeline=None, ...):
    pipeline = pipeline or Pipeline.get(pipeline_uuid, ...)

    if pipeline.variables is not None:
        # 优先从 Pipeline YAML 内存中读取
        return pipeline.variables
    else:
        # 兜底从 VariableManager 磁盘读取 (block_uuid='global')
        variables = variable_manager.get_variables_by_block(pipeline_uuid, 'global')
        global_variables = {}
        for variable in variables:
            global_variables[variable] = variable_manager.get_variable(
                pipeline_uuid, 'global', variable
            )
        return global_variables
```

**判断优先级**：
1. `Pipeline.variables` 不为 None → 直接返回 YAML 中的值
2. `Pipeline.variables` 为 None → 从磁盘 variables_dir 读取 `global/` 目录

### 4.2 单个读取：get_global_variable()

[variable_manager.py get_global_variable()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L544-L564)

```
pipeline.variables is not None?
  → pipeline.variables.get(key)          ← YAML
  → VariableManager.get_variable(pipeline_uuid, 'global', key)  ← 磁盘
```

### 4.3 写入：set_global_variable()

[variable_manager.py set_global_variable()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L580-L596)

```python
def set_global_variable(pipeline_uuid, key, value, repo_path=None):
    pipeline = Pipeline.get(pipeline_uuid, repo_path=repo_path)
    if pipeline.variables is None:
        pipeline.variables = get_global_variables(pipeline_uuid)
    pipeline.update_global_variable(key, value)
```

[Pipeline.update_global_variable()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/pipeline.py#L2209-L2217)：

```python
def update_global_variable(self, key, value):
    if not is_yaml_serializable(key, value):
        raise SerializationError(...)
    if self.variables is None:
        self.variables = {}
    self.variables[key] = value
    self.save()   # ← 写入 pipeline.yaml
```

**关键**：全局变量写入时 **同步持久化到 pipeline.yaml**，保证代码仓库版本控制。

### 4.4 删除：delete_global_variable()

[variable_manager.py delete_global_variable()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L599-L616)

```
pipeline.variables is not None?
  → pipeline.delete_global_variable(key)  → del self.variables[key]; self.save()
  → VariableManager.delete_variable(pipeline_uuid, 'global', key)  → 磁盘删除
```

---

## 五、API 层 CRUD 映射

[VariableResource.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/api/resources/VariableResource.py) 是前端与变量交互的唯一入口：

| HTTP | API 方法 | 内部调用 | 说明 |
|------|---------|---------|------|
| GET | `collection()` | `get_global_variables()` + `VariableManager.get_variables_by_pipeline()` | 列出 pipeline 全局 + Block 中间变量 |
| POST | `create()` | `set_global_variable()` | 创建全局变量（校验 isidentifier） |
| PUT/PATCH | `update()` | `set_global_variable()` | 更新全局变量 |
| DELETE | `delete()` | `delete_global_variable()` | 删除全局变量 |

**collection() 返回结构**：

```json
[
  {
    "block": {"uuid": "data_loader"},
    "pipeline": {"uuid": "abc123"},
    "variables": [
      {"uuid": "output_0", "type": "pandas.DataFrame", "value": "DataFrame"}
    ]
  },
  {
    "block": {"uuid": "global"},
    "pipeline": {"uuid": "abc123"},
    "variables": [
      {"uuid": "api_key", "type": "<class 'str'>", "value": "sk-xxx"}
    ]
  }
]
```

注意：DataFrame 类型变量在列表中只显示 `"DataFrame"` 字符串，不返回完整数据。

---

## 六、模板变量注入层

[shared/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/shared/utils.py) 提供了 metadata.yaml 和 io_config.yaml 渲染时的函数注入：

### 6.1 get_template_vars() — 含 DB 依赖

```python
def get_template_vars(include_python_libraries=None):
    no_db_kwargs = get_template_vars_no_db(...)
    kwargs = dict(mage_secret_var=get_secret_value)  # ← 需要 DB
    return merge_dict(no_db_kwargs, kwargs)
```

### 6.2 get_template_vars_no_db() — 无 DB 依赖

```python
def get_template_vars_no_db(include_python_libraries=None):
    kwargs = dict(
        env_var=os.getenv,            # {{ env_var('KEY') }}
        json_value=get_json_value,     # {{ json_value('...') }}
    )
    # 可选注入
    kwargs['aws_secret_var'] = get_secret    # {{ aws_secret_var('name') }}
    kwargs['azure_secret_var'] = get_secret  # {{ azure_secret_var('name') }}
    return kwargs
```

**使用场景**：

| 渲染目标 | 注入函数 | 说明 |
|---------|---------|------|
| `metadata.yaml` | `get_template_vars()` | 仓库配置加载时渲染 |
| `io_config.yaml` | `get_template_vars_no_db()` | IO 配置可能无 DB |
| `RepoConfig` 初始化 | `get_template_vars()` | Jinja2 模板渲染 |

---

## 七、Pipeline 执行链路中的变量合并

### 7.1 变量合并核心函数

[hash.py merge_dict()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/shared/hash.py#L198-L209)：

```python
def merge_dict(a: Dict, b: Dict) -> Dict:
    c = a.copy() if a else {}
    if not b:
        return c
    c.update(b)   # ← b 覆盖 a
    return c
```

**关键语义**：`merge_dict(a, b)` 中 **b 的值覆盖 a 的同名 key**。这是理解所有优先级冲突的核心。

### 7.2 PipelineRun.get_variables() — 完整合并链（按代码实际顺序）

[schedules.py PipelineRun.get_variables()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1534-L1660)

**代码实际执行顺序（从先到后，后者覆盖前者）：**

```python
def get_variables(self, extra_variables=None, pipeline_uuid=None):
    if extra_variables is None:
        extra_variables = dict()

    # === 第 1 步: 基础三层合并 (merge_dict 覆盖) ===
    # 全局变量 → 调度器变量 → 运行实例变量
    variables = merge_dict(
        merge_dict(
            get_global_variables(pipeline_uuid) or {},    # ① Pipeline YAML 全局变量
            self.pipeline_schedule.variables or {},        # ② Schedule 调度器变量 (覆盖①)
        ),
        self.variables or {},                              # ③ PipelineRun 运行实例变量 (覆盖②)
    )

    # === 第 2 步: 事件变量 (填空模式，不覆盖已有 key) ===
    for k, v in (self.event_variables or {}).items():
        if k not in variables:
            variables[k] = v

    # === 第 3 步: 系统注入变量 (直接赋值，覆盖所有前面) ===
    if self.execution_date:
        variables['ds'] = self.execution_date.strftime('%Y-%m-%d')
        variables['hr'] = self.execution_date.strftime('%H')

    variables['env'] = ENV_PROD
    variables['event'] = merge_dict(variables.get('event', {}), event_variables)  # event 字段内部是覆盖
    variables['execution_date'] = self.execution_date
    variables['execution_partition'] = self.execution_partition
    variables['pipeline_run_id'] = self.id
    variables['trigger_name'] = self.pipeline_schedule.name

    # === 第 4 步: 时间区间变量 (覆盖系统变量) ===
    # 根据 schedule_type 和 interval 计算 interval_end_datetime / interval_seconds 等
    # 同样直接赋值，覆盖前面

    # === 第 5 步 (最后!): extra_variables 额外变量 (覆盖一切!) ===
    variables.update(extra_variables)   # ← 最后一步，优先级最高

    return variables
```

**关键发现**：`extra_variables` 在函数**最后一行**通过 `variables.update(extra_variables)` 应用，它可以覆盖 **所有** 前面的变量，包括系统变量 `ds`、`hr`、`env`、`pipeline_run_id` 等。

### 7.3 两种运行时变量机制

Mage 中有 **两套独立的** 运行时变量传递机制：

| 机制 | 存储位置 | 作用时机 | 优先级 |
|------|---------|---------|--------|
| `pipeline_run.variables` | DB `pipeline_run.variables` JSON 列 | PipelineRun 创建时写入 | 第 3 级（低） |
| `extra_variables` | 函数参数，不持久化 | `get_variables()` 调用时传入 | 第 5 级（最高） |

**使用场景：**
- API 创建 PipelineRun：payload 中的 `variables` 存入 `pipeline_run.variables` → 第 3 级
- CLI `mage run --runtime-vars`：作为 `extra_variables` 传入 `get_variables()` → 第 5 级（最高）
- 调度器触发：变量来自 `pipeline_schedule.variables` → 第 2 级

### 7.4 变量合并优先级全景（Pipeline 级别）

从低到高，**后者覆盖前者**（基于代码实际执行顺序验证）：

```
┌────────────────────────────────────────────────────────────────────┐
│ 优先级 1 (最低)  Pipeline YAML 全局变量                            │
│   来源: pipeline.yaml → variables 字段                             │
│   读取: get_global_variables(pipeline_uuid)                        │
│   存储: 与代码同版本控制                                             │
│   合并: merge_dict 第一层                                           │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 2         PipelineSchedule 调度器变量                        │
│   来源: DB pipeline_schedule.variables (JSON 列)                   │
│   读取: self.pipeline_schedule.variables                           │
│   场景: 同一 pipeline 不同 trigger 可设不同变量                      │
│   合并: merge_dict 第二层 (覆盖优先级 1)                            │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 3         PipelineRun 运行实例变量 (DB 存储)                 │
│   来源: DB pipeline_run.variables (JSON 列)                        │
│   读取: self.variables                                             │
│   场景: API 创建 PipelineRun 时 payload.variables 存入 DB            │
│   合并: merge_dict 第三层 (覆盖优先级 2)                            │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 4         Event 事件变量 (填空模式)                          │
│   来源: DB pipeline_run.event_variables (JSON 列)                  │
│   规则: for k in event_variables: if k not in variables: set       │
│   特点: 仅补缺失，不覆盖已有 key (唯一例外)                          │
│   注意: 但 variables['event'] 字段用 merge_dict (event_variables    │
│         会覆盖 event 字典内部的同名 key)                             │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 5         系统注入变量 (直接赋值)                             │
│   注入: ds, hr, env, execution_date, execution_partition,          │
│         pipeline_run_id, trigger_name,                             │
│         interval_end_datetime, interval_start_datetime,            │
│         interval_seconds, interval_start_datetime_previous         │
│   特点: 直接 = 赋值，覆盖优先级 1-4 的同名 key                       │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 6 (最高)  extra_variables 额外运行时变量                      │
│   来源: get_variables(extra_variables=...) 函数参数                 │
│   合并: variables.update(extra_variables) ← 最后一步!               │
│   场景: CLI --runtime-vars / 运行时动态传入                          │
│   特点: 不持久化到 DB，仅本次调用生效                                 │
│  ⚠️ 可以覆盖系统变量! (ds, hr, env, pipeline_run_id 等)               │
└────────────────────────────────────────────────────────────────────┘
```

### 7.5 Block 级别变量覆盖（Pipeline 级别之上）

Block 执行时还有 **两层额外覆盖**，在 Pipeline 级 `global_vars` 基础上继续叠加：

```
Pipeline 级 global_vars (优先级 1-6 合并结果)
        ↓
  ┌──── 优先级 7: Hook 变量 ────┐
  │ 来源: block_run.metrics['hook_variables']
  │ 位置: BlockExecutor.execute() 第 178 行
  │ 合并: global_vars = merge_dict(global_vars, hook_variables)
  │ 条件: FeatureUUID.GLOBAL_HOOKS 启用
  └──────────────────────────────┘
        ↓
  ┌──── 优先级 8 (最高): 上游 Block kwargs 输出 ────┐
  │ 来源: 上游 Block 的输出变量中被标记为 kwargs 的部分
  │ 位置: Block.execute_sync() 第 1904-1908 行
  │ 合并: for kwargs_var in kwargs_vars:
  │         global_vars_copy.update(kwargs_var)
  │ 特点: 每个上游 Block 依次覆盖，后执行的上游优先级更高
  └──────────────────────────────────────────────┘
```

### 7.6 优先级冲突示例（同名 key `env` 在各层的覆盖）

| 层级 | 值 | 是否生效 | 说明 |
|------|---|---------|------|
| Pipeline YAML | `'dev'` | ❌ | 被调度器变量覆盖 |
| Schedule 变量 | `'staging'` | ❌ | 被 Run 变量覆盖 |
| PipelineRun 变量 | `'test'` | ❌ | 被系统变量覆盖 |
| Event 变量 | `'prod_event'` | ❌ | 填空模式，key 已存在不覆盖 |
| 系统注入 (`env = ENV_PROD`) | `'production'` | ❌ | 被 extra_variables 覆盖 |
| extra_variables (CLI runtime) | `'custom_env'` | ✅ | 最终生效（优先级 6） |
| Hook 变量 | `'hook_env'` | ✅ | 如果启用 Hook，覆盖上面所有 |
| 上游 kwargs 输出 | `'upstream_env'` | ✅ | 最终最终生效（优先级 8） |

**关键结论**：
1. `merge_dict(a, b)` 和 `dict.update(b)` 都是 **后者赢**（b 覆盖 a）
2. Event 变量是唯一 **不覆盖** 的（仅填空），但 `event` 字段内部是覆盖模式
3. 系统变量 **不是** 最高优先级，`extra_variables` 可以覆盖它们
4. Block 级别还有两层额外覆盖：Hook 变量 → 上游 kwargs 输出
5. 有 **两套独立的** 运行时变量：`pipeline_run.variables`（DB存，优先级3）和 `extra_variables`（参数传，优先级6）

---

## 八、执行链路时序图

### 8.1 Pipeline 执行（有 PipelineRun）

```
PipelineExecutor.execute(global_vars)
  ↓
pipeline_run_id is None?
  ├─ None → pipeline.execute(global_vars=global_vars)  ← 无 DB 记录
  └─ 有值  → PipelineRun.query.get(pipeline_run_id)
             → self.__run_blocks(pipeline_run, global_vars=global_vars)

  ↓ Pipeline.execute()
  → run_blocks(root_blocks, global_vars=global_vars, ...)
    → 递归执行每个 block（拓扑排序）
    → block.execute_sync(global_vars=global_vars, ...)

  ↓ Block.execute_sync()
  → fetch_input_variables(pipeline, upstream_block_uuids, global_vars=global_vars)
    → 从上游 Block 的 VariableManager 读取输出数据
  → global_vars_copy = global_vars.copy()
    → kwargs_var 是 dict? → global_vars_copy.update(kwargs_var)
  → self._execute_block(input_vars, global_vars=global_vars_copy, ...)

  ↓ Block._execute_block()
  → block_function(*input_vars, **global_vars)  ← 全局变量作为 **kwargs 注入
  → 输出结果 → VariableManager.add_variable() 写入磁盘
```

### 8.2 Block 执行（变量注入方式）

[block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/__init__.py) 中变量注入逻辑：

```python
# 如果 block 函数签名有 **kwargs，则注入全局变量
if has_kwargs and global_vars is not None and len(global_vars) != 0:
    output = block_function(*input_vars, **global_vars)
else:
    output = block_function(*input_vars)
```

**Sensor Block 特殊处理**：

```python
# Sensor 使用 global_vars 作为条件判断参数
condition = block_function(*args, **global_vars) if use_global_vars else block_function()
```

### 8.3 CLI `mage run` 的变量合并

[cli/main.py run()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/cli/main.py#L160-L276)：

```
1. parse_runtime_variables(runtime_vars)  ← 解析 CLI 传入的运行时变量
2. pipeline_run_id is None?
   → default_variables = get_global_variables(pipeline_uuid)  ← 仅全局变量
   → global_vars = merge_dict(default_variables, runtime_variables)  ← CLI 覆盖
3. pipeline_run_id exists?
   → pipeline_run.get_variables(extra_variables=runtime_variables)  ← DB 完整合并链
4. ExecutorFactory.get_pipeline/block_executor(pipeline, ...).execute(global_vars=global_vars)
```

---

## 九、fetch_input_variables — Block 间数据传递

[block/utils.py fetch_input_variables()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/utils.py#L389-L502) 负责从上游 Block 读取输出数据作为当前 Block 的输入：

```
1. input_args is not None?
   → input_vars = input_args  ← 直接传入（Notebook 交互模式）
2. pipeline is not None?
   → input_vars = [None] * len(upstream_block_uuids)
   → 对每个上游 block:
     a. cache_block_output_in_memory + 缓存命中? → 从内存缓存读取
     b. pipeline.get_block_variable(
          upstream_block_uuid, variable_uuid,
          global_vars=global_vars,    ← 传递全局变量（用于 data integration）
          partition=execution_partition,
        )
     c. get_block_variable 内部:
        → VariableManager.get_variable(pipeline_uuid, block_uuid, variable_uuid)
        → Variable.read_data() → 从磁盘/S3/GCS 读取
```

**global_vars 在 fetch_input_variables 中的作用**：

- 不直接参与上游数据读取
- 仅传给 `get_data_integration_settings()` 用于数据集成 Block 的配置渲染
- 上游 Block 的输出数据读取完全依赖 VariableManager

---

## 十、配置优先级冲突时的生效规则总结

### 10.1 全局变量 vs 调度器变量

```python
merge_dict(get_global_variables(), pipeline_schedule.variables)
```
→ **调度器变量覆盖全局变量**。同一 pipeline 绑定不同 trigger 时，trigger 变量可以覆盖 pipeline 默认值。

### 10.2 调度器变量 vs 运行实例变量（DB 存储）

```python
merge_dict(上面的结果, pipeline_run.variables)
```
→ **运行实例变量覆盖调度器变量**。API 创建 PipelineRun 时传入 `payload.variables` 存入 DB，在 merge_dict 第三层生效。

### 10.3 运行实例变量 vs 事件变量

```python
for k, v in event_variables.items():
    if k not in variables:
        variables[k] = v
```
→ **事件变量不覆盖**。仅当 key 不存在时才写入。这是一种"填空"语义。

⚠️ 例外：`variables['event']` 字段内部用 `merge_dict(variables.get('event', {}), event_variables)`，event 字典内部 event_variables 会覆盖。

### 10.4 系统变量 vs 前四层

系统变量（`ds`, `hr`, `env`, `pipeline_run_id`, `execution_partition`, `trigger_name`, `interval_*`）通过直接 `variables['key'] = value` 赋值，**覆盖优先级 1-4 的同名 key**。

### 10.5 extra_variables (runtime) vs 系统变量 ⭐ 最关键

```python
variables.update(extra_variables)  # 最后一行!
```
→ **extra_variables 覆盖一切**，包括系统变量。这是 `get_variables()` 函数的最后一步。

常见误解纠正：
- ❌ 错误：系统变量不可被覆盖
- ✅ 正确：`extra_variables`（CLI `--runtime-vars`）在系统变量之后应用，可以覆盖 `ds`、`env`、`pipeline_run_id` 等任何系统变量

### 10.6 Hook 变量 vs Pipeline 级全局变量

```python
# BlockExecutor.execute() 第 178-181 行
global_vars = merge_dict(global_vars, hook_variables)
```
→ **Hook 变量覆盖 pipeline 级所有变量**。在 BlockExecutor 入口处 merge，条件是 FeatureUUID.GLOBAL_HOOKS 启用。

### 10.7 上游 Block kwargs 输出 vs Hook 变量

```python
# Block.execute_sync() 第 1904-1908 行
global_vars_copy = global_vars.copy()
for kwargs_var in kwargs_vars:
    if kwargs_var:
        global_vars_copy.update(kwargs_var)
```
→ **上游 kwargs 输出覆盖 Hook 变量**。这是 Block 级别最高优先级，每个上游 Block 依次覆盖，后执行的上游优先级更高。

### 10.8 metadata.yaml 模板渲染 vs 全局变量

metadata.yaml 中的 `{{ env_var('KEY') }}` 和 `{{ mage_secret_var('key') }}` 在 RepoConfig 初始化时渲染，**与 pipeline 全局变量是完全独立的体系**：
- 模板变量用于配置渲染（DB 连接、IO 配置等）
- 全局变量用于 pipeline 执行时的 `**kwargs` 注入
- 两者不存在覆盖关系

### 10.9 两套运行时变量机制对比

| 机制 | 存储 | 优先级 | 传递方式 | 典型场景 |
|------|------|--------|---------|---------|
| `pipeline_run.variables` | DB JSON 列 | 第 3 级 | API 创建 PipelineRun 时存入 | API 触发、调度器触发 |
| `extra_variables` | 函数参数（不持久化） | 第 6 级（最高） | `get_variables(extra_variables=...)` | CLI `--runtime-vars`、内部调用 |

如果同时存在两种运行时变量，`extra_variables` 优先级更高，会覆盖 `pipeline_run.variables`。

---

## 十一、关键代码索引

| 功能 | 文件 | 关键行 |
|------|------|--------|
| VariableManager CRUD | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py) | L33-L481 |
| Variable 底层读写 | [variable.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/variable.py) | L70-L927 |
| VariableType 枚举 | [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/variables/constants.py) | L17-L31 |
| get_global_variables | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L504-L541) | L504-L541 |
| set_global_variable | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L580-L596) | L580-L596 |
| delete_global_variable | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/variable_manager.py#L599-L616) | L599-L616 |
| Pipeline.update_global_variable | [pipeline.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/pipeline.py#L2209-L2217) | L2209-L2217 |
| **PipelineRun.get_variables 完整合并链** | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1534-L1660) | **L1534-L1660** |
| **extra_variables 最后 update** | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1658-L1659) | **L1658** |
| 事件变量填空模式 | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1556-L1558) | L1556-L1558 |
| merge_dict (覆盖语义) | [hash.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/shared/hash.py#L198-L209) | L198-L209 |
| VariableResource API | [VariableResource.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/api/resources/VariableResource.py) | L1-L190 |
| get_template_vars | [shared/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/shared/utils.py#L7-L37) | L7-L37 |
| fetch_input_variables | [block/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/utils.py#L389-L502) | L389-L502 |
| Block **kwargs 注入 | [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2161-L2171) | L2161-L2171 |
| **Hook 变量 merge** | [block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L177-L181) | **L177-L181** |
| **上游 kwargs 输出 merge** | [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1904-L1908) | **L1904-L1908** |
| CLI run 变量流（两种分支） | [cli/main.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/cli/main.py#L237-L242) | L237-L242 |
| configure_pipeline_run_payload | [pipeline_scheduler_original.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py#L1365-L1394) | L1365-L1394 |

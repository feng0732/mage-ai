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

### 7.2 PipelineRun.get_variables() — 完整合并链

[schedules.py PipelineRun.get_variables()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1534-L1594)：

```python
def get_variables(self, extra_variables=None, pipeline_uuid=None):
    pipeline_run_variables = self.variables or {}
    event_variables = self.event_variables or {}

    # 第 1 层合并: 全局变量 ← 调度器变量
    variables = merge_dict(
        merge_dict(
            get_global_variables(pipeline_uuid) or {},   # 全局变量 (YAML/磁盘)
            self.pipeline_schedule.variables or {},       # 调度器变量 (DB JSON)
        ),
        pipeline_run_variables,                           # 本次运行变量 (DB JSON)
    )

    # 第 2 层: 事件变量 (不覆盖已存在的 key)
    for k, v in event_variables.items():
        if k not in variables:
            variables[k] = v

    # 第 3 层: 注入系统变量
    variables['ds'] = execution_date.strftime('%Y-%m-%d')
    variables['hr'] = execution_date.strftime('%H')
    variables['env'] = ENV_PROD
    variables['event'] = merge_dict(variables.get('event', {}), event_variables)
    variables['execution_date'] = self.execution_date
    variables['execution_partition'] = self.execution_partition
    variables['pipeline_run_id'] = self.id
    variables['trigger_name'] = self.pipeline_schedule.name

    # 第 4 层: 合并 CLI/runtime 传入的额外变量
    # (由调用方在 execute() 时 merge_dict(global_vars, extra_variables))
```

### 7.3 变量合并优先级全景

从低到高，**后者覆盖前者**：

```
┌────────────────────────────────────────────────────────────────────┐
│ 优先级 1 (最低)  Pipeline YAML 全局变量                            │
│   来源: pipeline.yaml → variables 字段                             │
│   读取: get_global_variables(pipeline_uuid)                        │
│   存储: 与代码同版本控制                                             │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 2         PipelineSchedule 调度器变量                        │
│   来源: DB pipeline_schedule.variables (JSON 列)                   │
│   读取: self.pipeline_schedule.variables                           │
│   场景: 同一 pipeline 不同 trigger 可设不同变量                      │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 3         PipelineRun 运行实例变量                           │
│   来源: DB pipeline_run.variables (JSON 列)                        │
│   读取: self.variables                                             │
│   场景: API 触发时传入的变量覆盖                                     │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 4         Event 事件变量 (非覆盖模式)                        │
│   来源: DB pipeline_run.event_variables (JSON 列)                  │
│   规则: 仅当 key 不存在时才写入，不覆盖已存在的变量                   │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 5         系统注入变量 (硬编码)                               │
│   注入: ds, hr, env, event, execution_date,                        │
│         execution_partition, pipeline_run_id, trigger_name         │
│   特点: 始终存在，不可被覆盖                                        │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 6         CLI/触发器 runtime 变量                            │
│   来源: CLI --runtime-vars / API 触发参数                           │
│   合并: merge_dict(global_vars, runtime_variables)                 │
│   特点: 由调用方在 execute() 前合并到 global_vars                    │
├────────────────────────────────────────────────────────────────────┤
│ 优先级 7 (最高)  Hook 变量                                          │
│   来源: block_run.metrics['hook_variables']                        │
│   合并: merge_dict(global_vars, hook_variables)                    │
│   特点: 仅在 FeatureUUID.GLOBAL_HOOKS 启用时生效                    │
└────────────────────────────────────────────────────────────────────┘
```

### 7.4 优先级冲突示例

假设同名变量 `key1` 在多层级存在：

| 层级 | 值 | 最终结果 |
|------|---|---------|
| Pipeline YAML | `'from_yaml'` | ❌ 被覆盖 |
| Schedule 变量 | `'from_schedule'` | ❌ 被覆盖 |
| PipelineRun 变量 | `'from_run'` | ✅ 生效（若无 CLI 覆盖） |
| Event 变量 | `'from_event'` | ❌ 不覆盖（非覆盖模式） |
| CLI runtime | `'from_cli'` | ✅ 生效（覆盖 Run 变量） |

**结论**：`merge_dict` 是浅覆盖，后者赢。Event 变量是唯一例外——仅填空不覆盖。

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

```
merge_dict(get_global_variables(), pipeline_schedule.variables)
```
→ **调度器变量覆盖全局变量**。同一 pipeline 绑定不同 trigger 时，trigger 变量可以覆盖 pipeline 默认值。

### 10.2 调度器变量 vs 运行实例变量

```
merge_dict(上面的结果, pipeline_run.variables)
```
→ **运行实例变量覆盖调度器变量**。每次触发运行时传入的变量具有更高优先级。

### 10.3 运行实例变量 vs 事件变量

```python
for k, v in event_variables.items():
    if k not in variables:
        variables[k] = v
```
→ **事件变量不覆盖**。仅当 key 不存在时才写入。这是一种"填空"语义。

### 10.4 系统变量 vs 所有层级

系统变量（`ds`, `hr`, `env`, `pipeline_run_id`, `execution_partition`, `trigger_name`）在所有层级合并之后才注入，**始终存在且不可被用户变量覆盖**。

### 10.5 CLI/runtime 变量 vs DB 合并结果

```
merge_dict(pipeline_run.get_variables(), runtime_variables)
```
→ **CLI runtime 变量覆盖 DB 合并结果**。这是最高用户可控优先级。

### 10.6 metadata.yaml 模板渲染 vs 全局变量

metadata.yaml 中的 `{{ env_var('KEY') }}` 和 `{{ mage_secret_var('key') }}` 在 RepoConfig 初始化时渲染，**与 pipeline 全局变量是完全独立的体系**：
- 模板变量用于配置渲染（DB 连接、IO 配置等）
- 全局变量用于 pipeline 执行时的 `**kwargs` 注入
- 两者不存在覆盖关系

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
| PipelineRun.get_variables | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1534-L1594) | L1534-L1594 |
| merge_dict (覆盖语义) | [hash.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/shared/hash.py#L198-L209) | L198-L209 |
| VariableResource API | [VariableResource.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/api/resources/VariableResource.py) | L1-L190 |
| get_template_vars | [shared/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/shared/utils.py#L7-L37) | L7-L37 |
| fetch_input_variables | [block/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/utils.py#L389-L502) | L389-L502 |
| Block **kwargs 注入 | [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2161-L2171) | L2161-L2171 |
| CLI run 变量合并 | [cli/main.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/cli/main.py#L227-L243) | L227-L243 |

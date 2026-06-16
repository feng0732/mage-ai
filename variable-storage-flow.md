# Variable 存储后端代码阅读指南

---

## 0. 文件位置速查

| 层级 | 模块路径 | 核心职责 |
|------|----------|----------|
| 触发层 | `mage_ai/data_preparation/executors/block_executor.py` | 块执行调度，控制 `store_variables` 开关 |
| 块层 | `mage_ai/data_preparation/models/block/__init__.py` | `store_variables()` 变量存储入口 |
| 管理层 | `mage_ai/data_preparation/variable_manager.py` | 变量管理器 + 存储后端工厂 |
| 模型层 | `mage_ai/data_preparation/models/variable.py` | 单变量读写 + 类型分派 |
| 批量层 | `mage_ai/data/models/manager.py` | DataManager 批量/流式读写 |
| 动态块 | `mage_ai/data_preparation/models/block/dynamic/variables.py` | 动态父子块索引映射 |
| 存储抽象 | `mage_ai/data_preparation/storage/base_storage.py` | 存储接口抽象 |
| 存储实现 | `mage_ai/data_preparation/storage/local_storage.py` | 本地文件系统 |
| 存储实现 | `mage_ai/data_preparation/storage/s3_storage.py` | AWS S3 |
| 存储实现 | `mage_ai/data_preparation/storage/gcs_storage.py` | Google Cloud Storage |
| 常量定义 | `mage_ai/data_preparation/models/constants.py` | `VARIABLE_DIR = '.variables'` |
| 文件命名 | `mage_ai/data_preparation/models/variables/constants.py` | 各类型文件名常量 |

---

## 1. 整体数据流

```
BlockExecutor.execute()
    ↓
BlockExecutor._execute()
    │
    ├─ store_variables = not cache_block_output_in_memory
    ├─ 数据集成块 → store_variables = False
    └─ PipelineType.INTEGRATION → 不存变量
    ↓
Block.execute_sync(store_variables=...)
    ↓
Block.execute_block()          执行业务代码
    ↓
post_process_output()         封装为 {output_0: ..., output_1: ...}
    ↓
[store_variables 为 True] ?
    ↓ YES
Block.store_variables(variable_mapping)
    ↓
uuid_for_output_variables()   计算 block_uuid 路径
    ↓
VariableManager.add_variable(pipeline_uuid, block_uuid, uuid, data)
    ↓
infer_variable_type()         自动检测 VariableType
    ↓
Variable.__init__(storage=...)
    ↓
Variable.write_data(data)
    ├─ DataManager.write_sync() 可写? → 批量写
    ├─ DataFrame → __write_parquet()
    ├─ 复杂对象 → __save_complex_object + __write_json
    └─ 其他 → __should_save_object + __write_json
    ↓
[variable_type != SPARK_DATAFRAME] ?
    ↓ YES                         ↓ NO (Spark 跳过)
write_metadata()              写 type.json (type/types)
    ↓
__write_resource_usage()      写 resource_usage.json（所有类型，包括 Spark）
```

---

## 2. 存储目录层级与路径构建

### 2.1 常量定义

`mage_ai/data_preparation/models/constants.py` 第 29 行定义变量根目录：

```python
VARIABLE_DIR = '.variables'
```

`mage_ai/data_preparation/models/variables/constants.py` 定义文件名：

```python
METADATA_FILE                    = 'type.json'
RESOURCE_USAGE_FILE             = 'resource_usage.json'
JSON_FILE                       = 'data.json'
JSON_SAMPLE_FILE                = 'sample_data.json'
DATAFRAME_PARQUET_FILE          = 'data.parquet'
DATAFRAME_PARQUET_SAMPLE_FILE   = 'sample_data.parquet'
DATAFRAME_COLUMN_TYPES_FILE     = 'data_column_types.json'
```

### 2.2 完整目录树

```
{variables_dir}/
└─ pipelines/
   └─ {pipeline_uuid}/                      # variable_manager.py 第475-480行 pipeline_path()
      └─ .variables/                       # VARIABLE_DIR = '.variables'
         └─ {partition}/                   # 执行分区，如 20240101T000000
            ├─ {block_uuid}/              # 普通块 block_dir_name = clean_name(block.uuid)
            │  ├─ output_0/
            │  │  ├─ data.parquet          # Pandas/Polars DataFrame
            │  │  ├─ sample_data.parquet
            │  │  ├─ data_column_types.json
            │  │  ├─ type.json             # metadata，Spark 不写此文件
            │  │  └─ resource_usage.json   # 所有类型都写，包括 Spark
            │  ├─ output_1/
            │  │  ├─ data.json             # 基本类型/复杂类型
            │  │  ├─ sample_data.json
            │  │  ├─ type.json
            │  │  └─ resource_usage.json
            │  └─ output_2/
            │     └─ ...
            │
            ├─ {block_uuid}/               # 动态子块（两种路径模式见 3.1）
            │  ├─ 0/                       # dynamic_block_index = 0
            │  │  └─ output_0/
            │  │     ├─ data.parquet
            │  │     ├─ type.json
            │  │     └─ resource_usage.json
            │  ├─ 1/
            │  │  └─ output_0/
            │  │     └─ ...
            │  └─ ...
            │
            └─ global/                     # 全局变量块
               └─ {var_name}/
```

### 2.3 路径构建代码链

1. **pipeline_path** — variable_manager.py 第 475-480 行：
   ```
   {variables_dir}/pipelines/{pipeline_uuid}
   ```

2. **variable_dir_path** — variable.py 第 105-110 行：
   ```
   pipeline_path + .variables + {partition} + block_dir_name
   ```
   `block_dir_name = clean_name(block_uuid) if clean_block_uuid else block_uuid`

3. **variable_path** — variable.py 第 137-138 行：
   ```
   variable_dir_path + {variable_uuid}
   ```

4. **metadata_path** — variable.py 第 141-142 行：
   ```
   variable_path + type.json
   ```

5. **resource_usage_path** — variable.py 第 144-147 行：
   ```
   variable_path + [index/] + resource_usage.json
   ```

---

## 3. 动态块父子索引与目录对应关系

### 3.1 两类动态路径模式

`mage_ai/data_preparation/models/block/dynamic/variables.py` 第 307-363 行 `get_variable_objects()` 的 docstring 明确了两种目录模式：

#### 模式 A：动态父块（上游是动态块）

变量输出按 `output_N` 下按数字子目录分片：

```
{block_dir_name}/
   ├─ output_0/
   │  ├─ 0/               ← part_uuid，由 get_part_uuids() 扫描
   │  │  ├─ data.parquet
   │  │  └─ type.json
   │  │
   │  ├─ 1/
   │  │  └─ ...
   │  └─ type.json        ← 父级 type.json: type=ITERABLE + types=[...]
   └─ output_1/
```

**关键函数**：
- `get_part_uuids()` 在 `mage_ai/data_preparation/models/variables/summarizer.py` 第 302-310 行
- `__get_part_uuids()` 同文件第 328-336 行：扫描 variable_path 下纯数字目录名

#### 模式 B：动态子块（块本身是动态子块，有 dynamic_block_index）

`block_uuid` 拼接 `dynamic_block_index` 作为 `block_dir_name` 下一级目录：

```
{block_dir_name}/
   ├─ 0/                  ← dynamic_block_index = 0，由 uuid_for_output_variables 生成
   │  └─ output_0/
   │     ├─ data.parquet
   │     └─ type.json
   ├─ 1/                  ← dynamic_block_index = 1
   │  └─ output_0/
   │     └─ ...
   └─ ...
```

**关键函数**：
- `uuid_for_output_variables()` 在 `mage_ai/data_preparation/models/block/dynamic/utils.py` 第 160-179 行：
  - `is_dynamic_child` 且 `dynamic_block_index` 不为空 → 返回 `os.path.join(block.uuid, str(dynamic_block_index))`, `changed=True`
  - `changed=True` → `clean_block_uuid=False` → `block_dir_name` 不做 `clean_name` 直接使用路径拼接

### 3.2 动态子块索引列表

- `get_dynamic_child_block_indexes()` — `dynamic/variables.py` 第 268-288 行：
  - 调 `build_combinations_for_dynamic_child()` 获取组合数，生成 `[0, 1, 2, ... count-1]`

- `__get_all_variable_objects_for_dynamic_child()` — 同文件第 390-420 行：
  - 遍历每个索引，逐个索引下取 `get_variable_objects(block, dynamic_block_index=i)` 得到该子块输出列表

### 3.3 动态块变量读取的懒加载

```
LazyVariableController → LazyVariableSet → LazyVariable
```

- `LazyVariable.read_data()` — `dynamic/variables.py` 第 55-68 行
- 动态子块的数据按 `child_dynamic_block_index` 取模运算选择

### 3.4 动态子块写入前清理旧数据

`store_variables()` — `models/block/__init__.py` 第 3823-3829 行：

```python
if not skip_delete and is_dynamic_child:
    delete_variable_objects_for_dynamic_child(
        self,
        dynamic_block_index=...,
        execution_partition=...,
    )
```

`delete_variable_objects_for_dynamic_child()` — `dynamic/variables.py` 第 366-387 行：
- `ExportWritePolicy.APPEND` 以外的写策略会先 `variable_object.delete()`

---

## 4. VariableType 与序列化策略对应

| VariableType | 主文件 | 采样文件 | 辅助文件 | 写入方法 |
|--------------|--------|----------|----------|----------|
| `DATAFRAME` | `data.parquet` | `sample_data.parquet` | `data_column_types.json` | `__write_parquet()` |
| `POLARS_DATAFRAME` | `data.parquet` | — | `data_column_types.json` | `__write_polars_dataframe()` |
| `SPARK_DATAFRAME` | parquet 多文件目录 | — | — | `__write_spark_parquet()` |
| `GEO_DATAFRAME` | `data.shp` 系列 | `sample_data.shp` | — | `__write_geo_dataframe()` |
| `DICTIONARY_COMPLEX` | `data.json` | `sample_data.json` | `data_column_types.json` | `__save_complex_object` + `__write_json` |
| `LIST_COMPLEX` | `data.json` | `sample_data.json` | `data_column_types.json` | 同上 |
| `MATRIX_SPARSE` | `data.json` (序列化矩阵) | — | — | `__write_matrix_sparse` + `__write_json` |
| `SERIES_PANDAS` | `data.parquet` 或 `data.json` | — | `data_column_types.json` | `__write_series_pandas` |
| `MODEL_SKLEARN` | `model.joblib` | — | — | `__should_save_object` |
| `MODEL_XGBOOST` | `model.ubj` | — | — | 同上 |
| `CUSTOM_OBJECT` | `object.joblib` | — | — | 同上 |
| `ITERABLE` | `output_N/0/`, `output_N/1/` 子目录 | — | 子目录各自 `type.json` | DataManager 或循环 `add_variable` |
| 默认 (JSON) | `data.json` | `sample_data.json` | — | `__write_json()` |

---

## 5. metadata.json (type.json) 写回边界

### 5.1 写入条件

**同步写入** — `variable.py` 第 795-798 行：

```python
# Shared logic across most variable types
if self.variable_type != VariableType.SPARK_DATAFRAME:
    # Not write json file in spark data directory to avoid read error
    self.write_metadata()
```

**异步写入** — `variable.py` 第 885-887 行：

```python
if self.variable_type != VariableType.SPARK_DATAFRAME:
    # Not write json file in spark data directory to avoid read error
    self.write_metadata()
```

**写入边界**：
- ✅ **所有类型除 SPARK_DATAFRAME 外均写入 metadata**
- ❌ **SPARK_DATAFRAME 不写入**（避免 Spark 目录中出现 json 导致读错误）

### 5.2 metadata 写入内容

`write_metadata()` — `variable.py` 第 908-926 行：

```python
{
  "type": "<VariableType 值字符串>",
  "types": ["子类型列表，当 variable_types 非空时才写"]  // ITERABLE 类型使用
}
```

### 5.3 其他写入 metadata 的时机

1. **ITERABLE 嵌套循环 `add_variable` 场景**：
   `variable_manager.py` 第 148-149 行，遍历 DataGenerator 分片后：

   ```python
   variable.variable_type = VariableType.ITERABLE
   variable.write_metadata()
   ```

2. **`check_variable_type` 自动修正**：
   `variable.py` 第 236-248 行，检测到 `part_uuids` 子目录存在但缺少 metadata，自动修正类型为 `ITERABLE` 并写回

3. **`add_variable_types()` 单独写**：
   `variable_manager.py` 第 164-184 行，直接写 metadata 的 `types` 数组

---

## 6. resource_usage.json 写回边界

### 6.1 写入触发点

**统一调用点**（对所有类型生效，包括 SPARK_DATAFRAME）：

- 同步写完成后 — `variable.py` 第 800 行
- 异步写完成后 — `variable.py` 第 889 行

```python
# 在 write_metadata() 之后（或 Spark 跳过 metadata 之后）统一调用
self.__write_resource_usage()
```

**`__write_resource_usage()` 实现** — `variable.py` 第 948-951 行：

```python
def __write_resource_usage(self) -> None:
    if self.resource_usage:
        os.makedirs(self.variable_dir_path, exist_ok=True)
        self.storage.write_json_file(self.resource_usage_path(), self.resource_usage.to_dict())
```

**写入文件路径**：
```
variable_path / resource_usage.json
```

### 6.2 resource_usage 更新点汇总

| 场景 | 更新位置 | 更新字段 |
|------|----------|----------|
| 复杂对象写 column_types | `variable.py` 第 667-670 行 | `directory`, `size` |
| 复杂对象异步 | `variable.py` 第 684-687 行 | 同上 |
| 自定义 joblib 对象 | `variable.py` 第 696-699 行 | 同上 |
| DataManager 写入后（同步） | `variable.py` 第 748-751 行 | `directory`, `size`（从 `data_manager.resource_usage` 复制） |
| DataManager 写入后（异步） | `variable.py` 第 844-847 行 | 同上 |
| `__write_json` 同步 | `variable.py` 第 1072-1075 行 | `size`, `path` |
| `__write_json_async` | `variable.py` 第 1086-1089 行 | 同上 |

### 6.3 读取

`get_resource_usage()` — `variable.py` 第 185-200 行：
- 支持 `index` 参数读取分片子目录：`variable_path/{index}/resource_usage.json`

---

## 7. 存储后端工厂与切换

### 7.1 VariableManager.get_manager()

`variable_manager.py` 第 44-59 行：

```python
if variables_dir.startswith("s3://"):    # S3VariableManager → S3Storage
if variables_dir.startswith("gs://"):    # GCSVariableManager → GCSStorage
其他:                                    # VariableManager   → LocalStorage
```

### 7.2 三类存储实现差异

| 特性 | LocalStorage | S3Storage | GCSStorage |
|------|--------------|------------|-------------|
| `makedirs` | `os.makedirs` | 空操作（S3 无目录概念） | 上传空 blob 占位 |
| `listdir` | `os.listdir` | `client.listdir` 前缀匹配 | `bucket.list_blobs` 前缀匹配 |
| `path_exists` | `os.path.exists` | `list_objects` `max_keys=1` | `blob.exists` |
| `remove` | `os.remove` | `client.delete_objects` | `bucket.delete_blob` |
| `remove_dir` | `shutil.rmtree` | 同 `remove` | `list_blobs` + 逐个 `delete` |
| 异步 JSON | `aiofiles` | 同步实现（TODO） | 同步实现（TODO） |

---

## 8. 不触发变量存储的场景

### 8.1 `store_variables = False` 的三种情况

1. **内存缓存模式** — `block_executor.py` 第 830-833 行：
   ```python
   if cache_block_output_in_memory:
       store_variables = False
   ```

2. **数据集成源/目标块** — `block_executor.py` 第 907-908 行：
   ```python
   # Source/Destination 块输出通过 data_integration 专用机制持久化
   store_variables = False
   ```

3. **Integration Pipeline 类型** — `models/block/__init__.py` 第 1619 行判断：
   ```python
   if store_variables and pipeline.type != PipelineType.INTEGRATION
   ```

### 8.2 VariableManager.add_variable 中的特殊分支

`variable_manager.py` 第 109-115 行：
- `write_batch_settings.mode` 非 `APPEND` 时先 `variable.delete()` —— 写前删旧

---

## 9. 关键调用链精读指引

阅读起点：`BlockExecutor.execute()` → `_execute()` → `block.execute_sync()`

### 9.1 变量存储入口阅读顺序

1. `Block.store_variables()` — `models/block/__init__.py` 第 3776-3861 行
2. `VariableManager.add_variable()` — `variable_manager.py` 第 61-162 行
3. `Variable.write_data()` — `models/variable.py` 第 720-812 行
4. 分派看 `__write_data()` — 同上第 727-812 行
5. 最后 `write_metadata()` + `__write_resource_usage()`

### 9.2 变量读取阅读顺序

1. 上游取数起点：`fetch_input_variables()` / `get_outputs()`
2. `VariableManager.get_variable()` — `variable_manager.py` 第 326-365 行
3. `Variable.read_data()` — `models/variable.py` 第 385-419 行
4. 看 `__read_data()` 类型分派读

### 9.3 动态块阅读顺序

1. `uuid_for_output_variables()` — `dynamic/utils.py` 第 160 行起 → 看 `block_uuid` 拼接
2. `get_variable_objects()` — `dynamic/variables.py` 第 307 行起
3. `get_outputs_for_dynamic_child()` → `LazyVariableController`
4. `fetch_input_variables_for_dynamic_upstream_blocks()` → 索引计算

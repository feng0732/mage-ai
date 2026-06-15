# Variable 存储后端代码分析

## 一、架构总览

Variable 存储后端采用分层设计，主要负责 Pipeline 执行过程中 Block 输出变量的持久化存储和读取。整体架构如下：

```
BlockExecutor (触发执行)
    ↓
Block.execute_sync() (执行块逻辑)
    ↓
Block.store_variables() (存储变量入口)
    ↓
VariableManager.add_variable() (变量管理)
    ↓
Variable.write_data() (数据写入)
    ↓
DataManager.write_sync/write_async (批量数据处理)
    ↓
BaseStorage 抽象层
    ├─ LocalStorage (本地文件系统)
    ├─ S3Storage (AWS S3)
    └─ GCSStorage (Google Cloud Storage)
```

---

## 二、主要参与对象

### 2.1 BlockExecutor - 块执行器

**文件**: [block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/executors/block_executor.py)

**核心职责**:
- 负责 Pipeline 中单个 Block 的执行调度
- 控制变量存储的开关 (`store_variables` 参数)
- 处理执行前的条件判断、重试逻辑、动态块扩展

**关键代码点**:
- [L830-833](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L830-L833): 根据 `cache_block_output_in_memory` 决定是否持久化变量
- [L907-908](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L907-L908): 数据集成块特殊处理，不存储变量
- [L1154-1172](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L1154-L1172): 调用 `block.execute_sync()` 并传入 `store_variables` 参数

### 2.2 Block - 块模型

**文件**: [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py)

**核心职责**:
- 执行块的业务逻辑
- 后处理输出并封装为变量映射
- 提供变量存储的入口方法

**关键方法**:

| 方法 | 位置 | 职责 |
|------|------|------|
| `execute_sync()` | [L1436](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1436) | 同步执行块，处理输出并调用变量存储 |
| `store_variables()` | [L3776](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776) | 同步存储变量到持久化后端 |
| `store_variables_async()` | [L3863](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3863) | 异步存储变量到持久化后端 |
| `__store_variables_prepare()` | 内部 | 准备变量映射，处理覆盖逻辑和删除旧变量 |

**关键代码流程**:
1. [L1595-1614](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1595-L1614): 将块输出转换为 `output_0`, `output_1` 等变量映射
2. [L1616-1633](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1616-L1633): 检查 `store_variables` 标志，调用 `_store_variables_in_block_function`
3. [L3841-3851](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3841-L3851): 遍历变量映射，调用 `variable_manager.add_variable()`

### 2.3 VariableManager - 变量管理器

**文件**: [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py)

**核心职责**:
- 工厂模式创建不同存储后端的管理器实例
- 提供变量的增删改查 API
- 管理变量目录结构和分区

**关键类**:

| 类 | 位置 | 存储后端 |
|----|------|----------|
| `VariableManager` | [L33](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L33) | 本地文件系统 (默认) |
| `S3VariableManager` | [L483](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L483) | AWS S3 |
| `GCSVariableManager` | [L491](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L491) | Google Cloud Storage |

**工厂方法**:
- [L44-59](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L44-L59): `get_manager()` 根据 `variables_dir` 前缀自动选择存储后端
  - `s3://` 前缀 → S3VariableManager
  - `gs://` 前缀 → GCSVariableManager
  - 其他 → VariableManager (LocalStorage)

**核心方法**:
- [L61-162](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L61-L162): `add_variable()` - 创建 Variable 对象并写入数据
- [L326-365](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L326-L365): `get_variable()` - 读取变量数据
- [L186-211](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L186-L211): `build_variable()` - 构建 Variable 对象用于读取

### 2.4 Variable - 变量模型

**文件**: [variable.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py)

**核心职责**:
- 封装单个变量的读写逻辑
- 根据变量类型选择不同的序列化方式
- 管理元数据和资源使用统计

**变量类型** (VariableType):
- `DATAFRAME` / `POLARS_DATAFRAME` / `SPARK_DATAFRAME` - 结构化数据，Parquet 格式
- `DICTIONARY_COMPLEX` / `LIST_COMPLEX` - 复杂对象，JSON + 列类型文件
- `MATRIX_SPARSE` - 稀疏矩阵，特殊序列化
- `MODEL_SKLEARN` / `MODEL_XGBOOST` - 机器学习模型
- `CUSTOM_OBJECT` - 自定义对象，Joblib 序列化
- `ITERABLE` - 可迭代对象，分块存储
- `JSON` - 基本类型，JSON 格式

**核心方法**:

| 方法 | 位置 | 职责 |
|------|------|------|
| `write_data()` | [L720](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L720) | 同步写入数据 |
| `__write_data()` | [L727](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L727) | 实际写入逻辑，根据类型分派 |
| `write_data_async()` | [L814](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L814) | 异步写入数据 |
| `read_data()` | [L385](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L385) | 同步读取数据 |
| `write_metadata()` | [L908](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L908) | 写入元数据 (类型信息) |

**关键写入流程**:
1. [L737-751](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L737-L751): 优先使用 DataManager 处理批量数据
2. [L771-793](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L771-L793): 根据变量类型选择写入方式
   - DataFrame → Parquet
   - 复杂对象 → JSON + 列类型文件
   - 其他 → JSON
3. [L796-800](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L796-L800): 写入元数据和资源使用统计

### 2.5 DataManager - 数据管理器

**文件**: [data/models/manager.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data/models/manager.py)

**核心职责**:
- 处理批量数据 (Batch) 和生成器 (Generator) 类型的输入输出
- 提供 Reader/Writer 抽象，支持分块读写
- 内存管理优化，避免大对象一次性加载

**关键属性**:
- `input_data_types`: 输入数据类型 (DEFAULT, BATCH, GENERATOR, READER)
- `read_batch_settings` / `write_batch_settings`: 批量读写配置

### 2.6 Storage 层 - 存储抽象

**基类**: [base_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/storage/base_storage.py)

**实现类**:

| 类 | 文件 | 存储介质 |
|----|------|----------|
| `LocalStorage` | [local_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/storage/local_storage.py) | 本地文件系统 |
| `S3Storage` | [s3_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/storage/s3_storage.py) | AWS S3 |
| `GCSStorage` | [gcs_storage.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/storage/gcs_storage.py) | Google Cloud Storage |

**核心接口**:
- `read_json_file()` / `write_json_file()` - JSON 读写
- `read_parquet()` / `write_parquet()` - Parquet 读写
- `read_json_file_async()` / `write_json_file_async()` - 异步 JSON 读写
- `path_exists()` / `isdir()` / `listdir()` - 路径操作
- `makedirs()` / `remove()` / `remove_dir()` - 目录操作

---

## 三、事件触发点

### 3.1 正常块执行流程

**触发链**:
```
1. BlockExecutor.execute()
   ↓ [L616-647]
2. __execute_with_retry()
   ↓ [L621-645]
3. BlockExecutor._execute()
   ↓ [L1154-1172]
4. Block.execute_sync()
   ↓ [L1570-1590]
5. Block.execute_block()  # 执行业务逻辑
   ↓ [L1595-1614]
6. post_process_output() → 生成 variable_mapping
   ↓ [L1616-1633]
7. _store_variables_in_block_function(variable_mapping)
   ↓ [L1536-1548]
8. Block.store_variables()
   ↓ [L3841-3851]
9. VariableManager.add_variable()
   ↓ [L151-152]
10. Variable.write_data()
```

### 3.2 动态块 (Dynamic Block) 特殊处理

**文件**: [block/dynamic/variables.py](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/dynamic/variables.py)

**关键触发点**:
- [L3823-3829](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3823-L3829): 动态子块执行前删除旧变量
- `get_outputs_for_dynamic_child()`: 获取动态子块的懒加载变量集合
- `fetch_input_variables_for_dynamic_upstream_blocks()`: 为动态子块获取上游输入

**存储路径结构**:
```
动态父块: output_0/0, output_0/1, output_0/2, ...
动态子块: {dynamic_block_index}/output_0, {dynamic_block_index}/output_1, ...
```

### 3.3 异步执行流程

**触发点**:
- `store_variables_async()`: 异步版本的变量存储
- `write_data_async()`: 异步数据写入
- 主要用于 Pipeline 调度器中的并行执行场景

### 3.4 不存储变量的场景

以下情况 `store_variables` 会被设置为 `False`:

1. **内存缓存模式**: [L830-833](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L830-L833)
   - `cache_block_output_in_memory=True` 时，变量只在内存中传递

2. **数据集成块**: [L907-908](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L907-L908)
   - Source/Destination 块的输出通过特殊机制处理，不经过通用变量存储

3. **Integration Pipeline**: [L1619](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1619)
   - `PipelineType.INTEGRATION` 类型的 Pipeline 不存储变量

---

## 四、结果回写机制

### 4.1 存储路径结构

**基础路径**:
```
{variables_dir}/pipelines/{pipeline_uuid}/.variables/
    ├─ {partition}/
    │   ├─ {block_uuid}/
    │   │   ├─ output_0/
    │   │   │   ├─ data.parquet        (DataFrame)
    │   │   │   ├─ sample_data.parquet (采样)
    │   │   │   ├─ column_types.json   (列类型)
    │   │   │   ├─ metadata.json       (变量类型)
    │   │   │   └─ resource_usage.json (资源使用)
    │   │   ├─ output_1/
    │   │   │   └─ data.json           (基本类型)
    │   │   └─ ...
    │   └─ global/                     (全局变量)
    └─ ...
```

**路径构建**:
- [variable_manager.py:L475-480](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L475-L480): `pipeline_path()` 构建 Pipeline 级路径
- [variable.py:L105-L112](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L105-L112): `variable_dir_path` 构建 Block 级变量目录
- [variable.py:L137-L138](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L137-L138): `variable_path` 构建单个变量目录

### 4.2 数据序列化策略

| 变量类型 | 存储格式 | 额外文件 |
|----------|----------|----------|
| DataFrame (Pandas) | Parquet | `column_types.json`, `sample_data.parquet` |
| DataFrame (Polars) | Parquet | `column_types.json` |
| DataFrame (Spark) | Parquet (目录) | - |
| GeoDataFrame | Shapefile | `sample_data.sh` |
| 复杂 Dict/List | JSON | `column_types.json` |
| 稀疏矩阵 | JSON (序列化后) | - |
| Sklearn 模型 | Joblib | - |
| XGBoost 模型 | UBJSON | - |
| 自定义对象 | Joblib | - |
| 基本类型 | JSON | `sample_data.json` |

### 4.3 元数据回写

**metadata.json 内容**:
```json
{
  "type": "dataframe",
  "types": ["dataframe", "dataframe"]  // ITERABLE 类型的子类型
}
```

**写入时机**:
- [variable.py:L796-798](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L796-L798): 除 Spark DataFrame 外，所有类型写入完成后调用 `write_metadata()`

### 4.4 资源使用统计回写

**resource_usage.json 内容**:
```json
{
  "directory": "/path/to/variable",
  "size": 1024000,
  "path": "/path/to/variable/data.parquet"
}
```

**写入时机**:
- [variable.py:L800](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/variable.py#L800): `__write_resource_usage()` 在数据写入后调用
- 各类型写入方法中更新 `resource_usage` 属性

### 4.5 旧数据清理

**触发点**:
- [variable.py:L114-L115](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L114-L115): 非 APPEND 模式下，写入前删除旧数据
- [block/__init__.py:L3853-3859](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3853-L3859): 存储新变量后删除 `removed_variables` 列表中的旧变量
- [variable_manager.py:L261-304](file:///d:/fz/0601/solo-dogfeeding/code/319-mage-ai/mage_ai/data_preparation/variable_manager.py#L261-L304): `clean_variables()` 定期清理超过保留期的变量

---

## 五、关键代码关系图

```
BlockExecutor
  ├─ execute()
  │   └─ _execute()
  │       └─ block.execute_sync(store_variables=...)
  │
Block
  ├─ execute_sync()
  │   ├─ execute_block() → 业务逻辑执行
  │   ├─ post_process_output() → output_0, output_1...
  │   └─ store_variables()
  │       ├─ __store_variables_prepare()
  │       ├─ delete_variable_objects_for_dynamic_child() (动态块)
  │       └─ variable_manager.add_variable() × N
  │
VariableManager
  ├─ get_manager() → 工厂方法选择存储后端
  ├─ add_variable()
  │   ├─ infer_variable_type()
  │   ├─ Variable.__init__()
  │   └─ variable.write_data()
  └─ get_variable()
      ├─ get_variable_object()
      └─ variable.read_data()
  │
Variable
  ├─ write_data()
  │   ├─ data_manager.write_sync() (批量数据)
  │   ├─ __write_parquet() (DataFrame)
  │   ├─ __write_json() (基本类型)
  │   ├─ write_metadata()
  │   └─ __write_resource_usage()
  └─ read_data()
      ├─ data_manager.read_sync() (批量数据)
      ├─ __read_parquet() (DataFrame)
      └─ __read_json() (基本类型)
  │
BaseStorage (抽象)
  ├─ LocalStorage
  ├─ S3Storage
  └─ GCSStorage
```

---

## 六、设计特点总结

1. **多态存储后端**: 通过 BaseStorage 抽象层，支持本地、S3、GCS 三种存储后端，通过路径前缀自动切换

2. **类型感知序列化**: 根据 VariableType 自动选择最优的序列化格式 (Parquet for DataFrame, JSON for primitives, etc.)

3. **懒加载支持**: LazyVariable/LazyVariableSet 实现按需加载，避免大对象一次性加载到内存

4. **批量处理优化**: DataManager 支持 BATCH/GENERATOR/READER 模式，支持流式处理和分块读写

5. **元数据驱动**: metadata.json 记录变量类型信息，支持动态类型推断和多类型集合

6. **资源监控**: 每个变量写入时记录资源使用情况 (文件大小、路径等)

7. **动态块支持**: 特殊的路径结构和删除逻辑支持动态父块/子块的变量存储

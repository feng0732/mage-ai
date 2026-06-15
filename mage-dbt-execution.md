# Mage AI 与 dbt 集成执行路径深度分析

## 一、总体架构概览

Mage AI 与 dbt 的集成采用 **进程内调用** 模式（非子进程），通过 `dbt-runner`（dbt 官方 Python SDK）在 Mage 进程内直接调用 dbt 命令。整体架构可划分为以下核心模块：

```
┌─────────────────────────────────────────────────────────────┐
│                     Mage AI 调度层                           │
│  Pipeline Scheduler → BlockExecutor → DBTBlock._execute_block │
└───────────┬─────────────────────────────────────┬───────────┘
            │                                     │
            ▼                                     ▼
┌───────────────────────┐           ┌──────────────────────────┐
│   DBTBlockSQL         │           │   DBTBlockYAML           │
│   (单模型执行)         │           │   (自由命令执行)          │
└───────────┬───────────┘           └───────────┬──────────────┘
            │                                     │
            ▼                                     ▼
┌──────────────────────────────────────────────────────────────┐
│                    DBTCli (命令编排层)                         │
│  dbtRunner.invoke() → 回调日志 → 结果解析                     │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│              Profiles (配置插值与隔离层)                       │
│  Jinja2 模板渲染 → 临时目录写入 → profiles-dir 指向临时路径     │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│           DBTAdapter (数据库连接适配层)                        │
│  RuntimeConfig → register_adapter → acquire_connection        │
└──────────────────────────────────────────────────────────────┘
```

---

## 二、模块职责详解

### 2.1 项目配置模块

#### 2.1.1 dbt 项目路径发现

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L70-L90)

`DBTBlockSQL.project_path` 属性通过向上遍历文件系统查找 `dbt_project.yml` / `dbt_project.yaml` 来确定 dbt 项目根目录：

```python
project_file_path = find_file_from_another_file_path(
    self.file_path,
    lambda x: os.path.basename(x) in ['dbt_project.yml', 'dbt_project.yaml'],
)
if project_file_path:
    return os.path.dirname(project_file_path)
```

**关键设计**：
- 每个 DBTBlock 持有自己的 `project_path`，一个 Mage pipeline 可包含来自不同 dbt 项目的 block
- `DBTBlock.base_project_path` 固定指向 `{repo_path}/dbt` 目录

#### 2.1.2 Profiles 配置插值

**关键文件**: [profiles.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/profiles.py#L29-L197)

`Profiles` 类实现了 **Jinja2 模板插值 → 临时文件写入 → 生命周期管理** 的完整流程：

1. **读取**：异步读取原始 `profiles.yml`
2. **插值**：使用 Jinja2 `Template` 渲染 Mage 变量（如 `{{ variables('key') }}`）
3. **临时写入**：在 `base_repo_path()` 下创建 `.profiles_interpolated_temp_{uuid}` 目录，写入插值后的 profiles
4. **清理**：通过 `__enter__`/`__exit__` 上下文管理器 + `__del__` 析构函数确保临时文件清理

```python
interpolated_profiles_dir = os.path.join(
    base_repo_path(),
    f'.profiles_interpolated_temp_{uuid.uuid4()}',
)
```

#### 2.1.3 Project 配置读取

**关键文件**: [project.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/project.py#L23-L113)

`Project` 类异步读取 `dbt_project.yml`，提供项目配置字典，同时提供 `local_packages` 属性发现本地 dbt 包。

#### 2.1.4 全局配置选项

**关键文件**: [configuration_option.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/settings/models/configuration_option.py#L40-L48)

`ConfigurationOption` 支持按 `ConfigurationType.DBT` 拉取项目配置，包括 `projects`、`profiles`、`targets` 三类选项，为前端 UI 提供配置下拉选项。

---

### 2.2 命令编排模块

#### 2.2.1 DBTCli 核心

**关键文件**: [dbt_cli.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_cli.py#L26-L132)

`DBTCli` 是 dbt 命令的统一封装层，核心方法 `invoke()`：

```python
dbt = dbtRunner(
    callbacks=[build_logging_callback(self.__log, log_level=log_level)],
)
res = dbt.invoke(cli_args)
```

**设计要点**：
- 使用 `dbtRunner`（dbt 官方 Python 接口），**非子进程调用**
- 自动补全 `--profiles-dir` 和 `--project-dir` 参数（如未显式指定）
- 日志通过 `callbacks` 机制实时回传
- `show` 方法封装 `dbt show` 命令用于数据预览
- `to_pandas` 方法将 `agate_table` 转为 `pd.DataFrame`

#### 2.2.2 SQL Block 执行流程

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L305-L440)

`DBTBlockSQL._execute_block()` 的完整执行流程：

```
1. __create_upstream_tables()     ← 物化上游 block 输出为 dbt seed
2. __task()                       ← 决定执行哪个 dbt 任务
3. 构建 CLI参数
   ├── --project-dir
   ├── runtime flags (--full-refresh等)
   ├── --select (模型选择)
   ├── --vars (变量JSON)
   ├── --target (可选)
   └── --profiles-dir
4. dbt deps                       ← 安装依赖
5. dbt {task}                     ← 执行主任务 (run/build/test/snapshot)
6. dbt show (可选)                 ← 获取预览/下游数据
7. store_variables()              ← 存储输出
```

#### 2.2.3 任务选择逻辑（__task 方法）

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L536-L586)

`__task(from_notebook, run_settings)` 方法根据两个输入参数决定执行哪个 dbt 命令。

**核心代码逻辑**：
```python
if from_notebook:
    if run_settings is not None:
        if run_settings.get('run_model'):
            return 'snapshot' if __node_type == 'snapshot' else 'run'
        elif run_settings.get('test_model'):
            return 'test'
        elif run_settings.get('build_model'):
            return 'build'
        else:
            return 'show'   # run_settings 为空字典时走这里
elif disable_tests:
    return 'snapshot' if __node_type == 'snapshot' else 'run'
return 'build'  # 后台调度默认 build
```

**run_settings 的重要转换（从 None 到 {}）**：

前端 Websocket 消息中 `run_settings` 字段可选（不传则为 `None`），但在生成内核执行代码时，[output_display.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/server/utils/output_display.py#L235) 中：
```python
run_settings_json = json.dumps(run_settings or {})
```
`run_settings or {}` 将 `None` 转换为空字典 `{}`，再经 `json.loads()` 解析后传递给 `_execute_block`。

**因此，所有来自 Notebook 前端的调用，`run_settings` 均不为 `None`**——要么是空字典 `{}`，要么是包含具体选项的字典。

**任务选择矩阵（实际行为）**：

| 触发源 | from_notebook | run_settings | disable_tests | dbt 任务 |
|-------|---------------|--------------|---------------|----------|
| Preview 按钮 (Cmd+Enter) | True | {} | any | show |
| Run 按钮 | True | {run_model:True} | any | run / snapshot |
| Test 按钮 | True | {test_model:True} | any | test |
| Build 按钮 | True | {build_model:True} | any | build |
| 后台调度 (PipelineRun) | False | 任意 | False/None | build |
| 后台调度 (PipelineRun) | False | 任意 | True | run / snapshot |

**关键洞察**：
- `Preview` 按钮 → `run_settings={}` → `show` 命令（仅查询不物化）
- `Run/Test/Build` 按钮 → 对应 `run_model/test_model/build_model` → 对应 dbt 命令
- `snapshot` 模型节点自动将 `run` 替换为 `snapshot` 命令
- 后台调度默认使用 `build`（内含 run + test），若配置了 `disable_tests` 则降级为 `run`

#### 2.2.4 前端触发路径与参数传递

**关键文件**: [useCodeBlockProps.tsx](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/frontend/components/CodeBlockV2/dbt/useCodeBlockProps.tsx#L149-L264)

dbt SQL Block 在 Notebook 界面提供 **4 个执行按钮**：

| 按钮 | 快捷键 | run_settings | 描述 |
|------|--------|--------------|------|
| Preview | Cmd/Ctrl + Enter | 无（→ {}) | 编译 SQL 并查询样本数据 |
| Run | - | {run_model: true} | 运行模型（物化表） |
| Test | - | {test_model: true} | 测试模型（执行 schema 测试） |
| Build | - | {build_model: true} | 构建模型（run + test） |

**完整调用链路**：
```
前端按钮点击
  └─ runBlockAndTrack(block, run_settings)
       └─ WebSocket 消息 { type, uuid, run_settings }
            └─ websocket_server.__execute_block()
                 └─ add_execution_code() 生成内核代码
                      ├─ run_settings or {}  → JSON 序列化
                      └─ IPython 内核执行 execute_custom_code()
                           └─ block.execute_with_callback(run_settings=...)
                                └─ block._execute_block(run_settings=...)
                                     └─ __task(from_notebook=True, run_settings=...)
```

#### 2.2.5 deps 触发条件

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L411)

`dbt deps` 的触发是 **无条件的**——每次执行 dbt block 都会先执行 `dbt deps` 安装依赖包。

```python
cli.invoke(['deps'] + args)   # 始终执行，无前置判断
```

**两种 Block 类型的一致性**：
- ✅ `DBTBlockSQL`：每次执行前都跑 `dbt deps`
- ✅ `DBTBlockYAML`：每次执行前都跑 `dbt deps`

**潜在影响**：
- 每次执行都触发包下载，可能拖慢执行速度
- 网络不稳定时 `deps` 失败会导致整个 block 失败
- 无缓存机制，重复下载相同的包

#### 2.2.6 show 触发条件与数据获取

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L382-L430)

`dbt show` 用于获取模型输出的 DataFrame，供预览或下游 block 消费。由两个独立的布尔变量控制：

| 变量 | 条件 | limit | 用途 |
|------|------|-------|------|
| `needs_preview_df` | `from_notebook=True` 且 `task != 'test'` | `configuration.get('limit', 1000)` | Notebook 内数据预览 |
| `needs_downstream_df` | `from_notebook=False` 且存在下游非 dbt block (Python/R/SQL) | -1（不限行数） | 下游 block 数据输入 |

**show 与主任务的执行顺序**：

```
情况 1：task == 'show'（Preview 按钮）
  └─ 跳过主任务（if task != 'show' 不满足）
  └─ 执行 dbt show → 获取 df

情况 2：task == 'run' / 'build' / 'snapshot'（Run/Build 按钮 + 非 test）
  ├─ 执行主任务（dbt run / build / snapshot）
  └─ needs_preview_df 或 needs_downstream_df 为 True → 再执行 dbt show → 获取 df

情况 3：task == 'test'（Test 按钮）
  ├─ 执行主任务（dbt test）
  └─ needs_preview_df = False（因为 task == 'test'）且 needs_downstream_df 通常为 False
  └─ 不执行 show → df = None
```

**重要注意**：
- `task == 'show'` 时只执行 `dbt show`，不物化模型（不修改数据库中的表）
- `task == 'run'/'build'` 时，若需要预览/下游数据，会**额外再执行一次** `dbt show`
- `task == 'test'` 时不执行 show，因为测试没有数据输出

**YAML Block 的差异**：
- ❌ `DBTBlockYAML`：**不执行** `dbt show`，不生成 DataFrame 输出
- 仅执行用户指定的 dbt 命令（默认 `run`），成功即返回

#### 2.2.7 YAML Block 执行流程（补充）

**关键文件**: [block_yaml.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_yaml.py#L92-L195)

`DBTBlockYAML._execute_block()` 允许用户在 block content 中编写自由格式的 dbt 命令：

```
1. 从配置获取 command (默认 'run')
2. Jinja2 插值 block 内容
3. shlex.split(content) 解析参数（支持引号处理）
4. 合并系统参数：--project-dir, flags, --vars, --target, --profiles-dir
5. dbt deps                          ← 始终执行
6. dbt {task} {user_args + sys_args}  ← 执行用户命令
```

**与 SQL Block 的核心差异**：
| 特性 | DBTBlockSQL | DBTBlockYAML |
|------|-------------|--------------|
| 上游 DataFrame 物化 | ✅ 自动处理 | ❌ 不处理 |
| dbt show 获取数据 | ✅ 自动执行 | ❌ 不执行 |
| 输出 DataFrame | ✅ df / output_0 | ❌ 无数据输出 |
| 依赖图解析 | ✅ 自动构建 | ❌ 不构建 |
| 命令灵活性 | 固定（run/test/build/show/snapshot） | 自由（任意 dbt 命令） |
| 适用场景 | 单模型精确控制 | 高级用户自由编排 |

---

### 2.3 运行环境模块

#### 2.3.1 进程内执行（无环境隔离）

Mage AI 的 dbt 执行 **在 Mage 主进程内** 进行，通过 `dbtRunner.invoke()` 直接调用。这意味着：

- **无 Python 虚拟环境隔离**：dbt 与 Mage 共享同一 Python 解释器和依赖空间
- **无容器级隔离**：不像 dbt Cloud 那样在独立容器中运行
- **profiles 隔离通过临时目录实现**：每次执行创建独立的 `.profiles_interpolated_temp_{uuid}` 目录

#### 2.3.2 dbt 适配器注册机制

**关键文件**: [dbt_adapter.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_adapter.py#L39-L261)

`DBTAdapter` 通过 dbt 官方适配器注册机制建立数据库连接：

```python
config = RuntimeConfig.from_args(adapter_config)
reset_adapters()
register_adapter(config, mp_context=get_mp_context())
self.__adapter = get_adapter(config)
self.__adapter.acquire_connection('mage_dbt_adapter_' + uuid.uuid4().hex)
```

**关键设计**：
- `reset_adapters()` 在每次 `open()` 时重置全局适配器注册表，**确保不同 block 间的适配器状态不泄漏**
- 连接名称使用 UUID 后缀 `mage_dbt_adapter_{hex}`，避免连接名冲突
- 使用上下文管理器 `__enter__`/`__exit__` 管理连接生命周期

#### 2.3.3 BlockFactory 动态分派

**关键文件**: [block_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/block_factory.py#L19-L33)

```python
elif BlockType.DBT == block_type:
    if language == BlockLanguage.YAML:
        return DBTBlockYAML
    return DBTBlockSQL
```

若 dbt 库未安装，import 异常被静默捕获：`except Exception: print('DBT library not installed.')`

---

### 2.4 日志搜集模块

#### 2.4.1 回调式日志捕获

**关键文件**: [dbt_cli.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_cli.py#L16-L23)

```python
def build_logging_callback(logging_func, log_level=None):
    def __callback(event, log_level=log_level, logging_func=logging_func):
        log_levels = set([LogLevel.INFO, LogLevel.WARN, LogLevel.ERROR])
        if LogLevel.DEBUG == log_level or event.info.level in log_levels:
            logging_func(event.info.level, event.info.msg)
    return __callback
```

**机制**：
- dbtRunner 的 `callbacks` 参数接收事件回调
- 每个事件包含 `event.info.level`（dbt 日志级别）和 `event.info.msg`（日志消息）
- 默认只记录 INFO/WARN/ERROR 级别；DEBUG 模式下记录全部
- 日志通过 Mage 的 `DictLogger` 或标准 `Logger` 输出

#### 2.4.2 BlockExecutor 日志管理

**关键文件**: [block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L76-L80)

```python
self.logger_manager = LoggerManagerFactory.get_logger_manager(
    pipeline_uuid=self.pipeline.uuid,
    block_uuid=clean_name(self.block_uuid),
    partition=self.execution_partition,
    repo_config=self.pipeline.repo_config,
)
```

BlockExecutor 为每个 block 创建独立的 `LoggerManager`，日志按 pipeline/block/partition 隔离存储。

#### 2.4.3 命令行格式化输出

**关键文件**: [dbt_cli.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_cli.py#L68-L83)

DBTCli 将执行的 dbt 命令格式化为可读的 shell 格式输出：

```python
message = 'dbt'
log_args = [message] + [str(a) for a in cli_args]
# 将参数两两配对，以 \ 连接换行展示
message = ' \\\n    '.join(pairs)
self.__info(message, tags)
```

---

### 2.5 结果反馈模块

#### 2.5.1 执行状态判断

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L414-L431)

SQL Block 中有 **两处** 执行状态判断：

1. **主任务执行后**（第 414-418 行）：
```python
if task != 'show':
    res = cli.invoke([task] + args)
    success = res.success
    if not success:
        raise Exception(str(res.exception))
```

2. **show 任务执行后**（第 426-430 行）：
```python
if needs_downstream_df or needs_preview_df:
    res = cli.invoke(['show'] + args)
    if res.success:
        df = cli.to_pandas(res)
    else:
        raise Exception(str(res.exception))
```

**错误处理行为**：
- `dbtRunnerResult.success` 为 `True` 表示执行成功
- 失败时抛出 `Exception(str(res.exception))`，异常信息直接来自 dbt 的异常对象
- **任何一步失败都会立即中断**，异常向上传播至 `BlockExecutor`
- `dbt deps` 执行后**不检查 success**——即使 deps 失败，也会继续执行主任务（潜在 bug）

**YAML Block 的一致性**：
- ✅ 同样使用 `raise Exception(str(res.exception))` 模式
- ✅ 失败时立即中断
- ❌ `dbt deps` 同样不检查 success

#### 2.5.2 数据输出与存储

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L433-L440)

```python
self.store_variables(
    {'df' if from_notebook else 'output_0': df},
    execution_partition=execution_partition,
    override_outputs=True,
)
return [df]
```

**存储变量的键名差异**：
| 执行模式 | 存储键 | 用途 | 消费者 |
|---------|--------|------|--------|
| Notebook (from_notebook=True) | `df` | 前端数据预览 | UI 展示层 |
| 后台调度 (from_notebook=False) | `output_0` | 下游 block 输入 | 后续 block |

**各执行路径的输出情况**：

| 执行路径 | 主任务 | show 执行 | df 结果 | 存储键 |
|---------|--------|-----------|---------|--------|
| Preview (show) | 无 | ✅ 是 | 预览数据（limit 行） | df |
| Run | ✅ run | ✅ 是（notebook 模式） | 完整结果预览（limit 行） | df |
| Test | ✅ test | ❌ 否 | None | df |
| Build | ✅ build | ✅ 是（notebook 模式） | 完整结果预览（limit 行） | df |
| 后台调度 | ✅ build/run | ✅ 是（有下游非 dbt block 时） | 完整数据（无行数限制） | output_0 |
| 后台调度 | ✅ build/run | ❌ 否（无下游非 dbt block 时） | None | output_0 |

**关键洞察**：
- `task == 'show'` 时，**不执行主任务**，只执行 show 进行数据预览（不物化模型）
- 后台调度时，`needs_downstream_df` 由**下游 block 的类型**决定——只有当下游存在非 dbt block 时，才会执行 show 获取 DataFrame
- 若 dbt pipeline 中所有下游都是 dbt block，则**不执行 show、不生成 DataFrame**，节省资源
- `limit = -1` 表示不限制行数（给下游的完整数据），预览模式默认 1000 行

#### 2.5.3 结果写回下游的完整链路

**dbt → 非 dbt 下游的数据流向**：

```
上游 dbt block (后台调度)
  └─ needs_downstream_df = True (检测到下游非 dbt block)
       └─ 执行 dbt show --limit -1
            └─ cli.to_pandas(res) → DataFrame
                 └─ store_variables({'output_0': df})
                      └─ VariableManager 存储到磁盘/内存
                           └─ 下游非 dbt block 读取 output_0
                                └─ 作为 DataFrame 输入继续执行
```

**上游非 dbt → dbt 的数据流向**（通过 source 桥接）：

```
上游非 dbt block
  └─ 输出 DataFrame
       └─ Sources.add_blocks() → 写入 mage_sources.yml
            └─ __create_upstream_tables()
                 └─ DBTBlock.materialize_df()
                      ├─ 写 CSV 到 seed-paths
                      ├─ dbt seed --full-refresh → 物化表
                      └─ 删除 CSV
                           └─ dbt 模型通过 {{ source() }} 引用该表
```

**双向数据流总结**：
- **入站**（非 dbt → dbt）：DataFrame → CSV → dbt seed → 数据库表 → dbt source 引用
- **出站**（dbt → 非 dbt）：dbt 模型 → dbt show → agate_table → DataFrame → output_0 变量

#### 2.5.4 上游 DataFrame 物化

**关键文件**: [block.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block.py#L149-L203)

`DBTBlock.materialize_df()` 将上游 block 的 DataFrame 物化为 dbt 可消费的形式：

1. 将 DataFrame 写入 CSV（`mage_{pipeline_uuid}_{block_uuid}.csv`）
2. 放置在 dbt 项目的 `seed-paths` 目录下
3. 执行 `dbt seed --select mage_{pipeline_uuid}_{block_uuid} --full-refresh`
4. 执行后删除 CSV 文件

物化后的表名为 `{default_database}.{default_schema}.mage_{pipeline_uuid}_{block_uuid}`

---

### 2.6 调度整合模块

#### 2.6.1 Block 创建与依赖图构建

**关键文件**: [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1024-L1035)

当创建 DBT SQL Block 时，Mage 自动解析 dbt 依赖图：

```python
if BlockType.DBT == block.type and block.language == BlockLanguage.SQL:
    upstream_dbt_blocks = block.upstream_dbt_blocks() or []
    upstream_dbt_blocks_by_uuid = {
        block.uuid: block for block in upstream_dbt_blocks
    }
    pipeline.blocks_by_uuid.update(upstream_dbt_blocks_by_uuid)
    pipeline.validate('A cycle was formed while adding a block')
    pipeline.save()
```

#### 2.6.2 dbt 依赖图解析

**关键文件**: [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L142-L303)

`upstream_dbt_blocks()` 方法通过 `dbt list` 命令获取上游依赖图：

1. 执行 `dbt list --select +{model_name} --output json --output-keys unique_id original_file_path depends_on`
2. 解析返回的 JSON 节点
3. 构建 upstream/downstream 关系图
4. 将 dbt `unique_id` 映射为 Mage block UUID
5. 创建 `DBTBlockSQL` 实例并建立上下游关系

**重要**：此操作在 block 创建时执行，将 dbt 内部依赖关系"提升"为 Mage pipeline 的 block 依赖关系。

#### 2.6.3 Sources 自动管理

**关键文件**: [sources.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/sources.py#L20-L267)

`Sources` 类自动管理 `mage_sources.yml`，实现 Mage 上游 block 与 dbt source 的桥接：

- `add_blocks()`: 将上游非 dbt block 注册为 dbt source table
- `cleanup_pipeline()`: 清理不再关联的 source table
- `reset_pipeline()`: 先 add 再 cleanup

生成的 source 条目格式：
```yaml
sources:
  - name: mage_{project_name}
    schema: {credentials.schema}
    tables:
      - name: {pipeline_uuid}_{block_uuid}
        identifier: mage_{pipeline_uuid}_{block_uuid}
        meta:
          pipeline_uuid: ...
          block_uuid: ...
```

#### 2.6.4 Pipeline 级 Sources 更新触发

**关键文件**: [pipeline.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/pipeline.py#L1538-L1553)

Pipeline 保存时，若存在非 dbt block → dbt block 的下游关系，自动触发 `DBTBlock.update_sources()`：

```python
if any(
    BlockType.DBT != block.type
    and block.language in [BlockLanguage.SQL, BlockLanguage.PYTHON, BlockLanguage.R]
    and any(
        BlockType.DBT == downstream_block.type
        for downstream_block in block.downstream_blocks
    )
    for _uuid, block in self.blocks_by_uuid.items()
):
    DBTBlock.update_sources(self.blocks_by_uuid, variables=self.variables)
```

#### 2.6.5 BlockExecutor 中的 dbt 特殊处理

**关键文件**: [block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/executors/block_executor.py#L1174-L1179)

```python
if BlockType.DBT == self.block.type:
    self.block.run_tests(...)  # 实际为 no-op，因为 dbt 内部处理测试
```

dbt block 的 `run_tests()` 被覆写为空操作，因为 dbt 在 `build` 或 `run+test` 命令中已内含测试逻辑。

#### 2.6.6 DBT Cache 机制

**关键文件**: [cache.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/cache/dbt/cache.py#L21-L177)

`DBTCache` 为前端提供 dbt 项目/模型/配置的缓存索引：

- `initialize_cache()` / `initialize_cache_async()`: 扫描所有 dbt 项目，构建映射
- `update()`: 文件变更时增量更新缓存（区分 project/profiles/model 三种文件类型）
- 缓存结构：`{project_path: {project: {...}, profiles: {...}, models: [...]}}`

---

### 2.7 dbt Cloud 集成

**关键文件**: [dbt.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/services/dbt/dbt.py#L13-L175)

`DbtCloudClient` 提供 dbt Cloud API 的封装：

- `list_jobs()`: 列出项目中的 job
- `list_runs()` / `get_run()`: 查询运行历史
- `trigger_job_run()`: 触发 job 运行，支持轮询等待完成

**轮询机制**：
```python
while True:
    run_data = self.get_run(run_id)['data']
    if job_run_status in DbtCloudJobRunStatus.TERMINAL_STATUSES.value:
        if job_run_status == DbtCloudJobRunStatus.SUCCESS.value:
            break
        raise Exception(...)
    time.sleep(poll_interval)
```

**状态枚举**: [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/services/dbt/constants.py#L8-L15)

| 状态 | 值 | 含义 |
|------|---|------|
| QUEUED | 1 | 排队中 |
| STARTING | 2 | 启动中 |
| RUNNING | 3 | 运行中 |
| SUCCESS | 10 | 成功 |
| ERROR | 20 | 失败 |
| CANCELLED | 30 | 已取消 |

---

### 2.8 命令编排与结果反馈一致性核对

#### 2.8.1 SQL Block 命令编排全景

**完整命令序列**（按执行顺序）：

```
步骤 0：__create_upstream_tables()
  └─ 对每个上游非 dbt block
       └─ DBTBlock.materialize_df()
            └─ dbt seed --select mage_xxx --full-refresh  ← 每上游 block 一次

步骤 1：dbt deps  ← 始终执行，不检查结果

步骤 2：主任务（条件执行）
  ├─ task == 'show' → 跳过
  ├─ task == 'run' → dbt run --select ...
  ├─ task == 'test' → dbt test --select ...
  ├─ task == 'build' → dbt build --select ...
  └─ task == 'snapshot' → dbt snapshot --select ...

步骤 3：dbt show（条件执行）
  └─ needs_preview_df 或 needs_downstream_df 为 True → 执行
```

**命令数统计**：
- 最少：2 条命令（deps + 主任务，无 show）
- 最多：N + 3 条命令（N 个上游 seed + deps + 主任务 + show）

#### 2.8.2 结果反馈一致性分析

| 检查点 | DBTBlockSQL | DBTBlockYAML | 一致? |
|--------|-------------|--------------|-------|
| dbt deps 执行 | ✅ 始终执行 | ✅ 始终执行 | ✅ 一致 |
| dbt deps 结果检查 | ❌ 不检查 success | ❌ 不检查 success | ✅ 一致（都不检查）|
| 主任务成功判断 | `res.success` | `res.success` | ✅ 一致 |
| 主任务失败处理 | `raise Exception(str(res.exception))` | `raise Exception(str(res.exception))` | ✅ 一致 |
| show 任务 | ✅ 条件执行 | ❌ 不执行 | ❌ 不一致 |
| DataFrame 输出 | ✅ df / output_0 | ❌ 无输出 | ❌ 不一致 |
| store_variables | ✅ 存储输出 | ❌ 不存储 | ❌ 不一致 |
| 返回值格式 | `[df]` (List) | None | ❌ 不一致 |

**一致性结论**：
- **错误处理模式一致**：两者都使用 `res.success` 判断 + `Exception(str(res.exception))` 抛出
- **deps 行为一致**：都不检查 deps 成功与否（可能是故意设计，也可能是 bug）
- **数据输出不一致**：SQL Block 有完整的 show + DataFrame 输出链路，YAML Block 没有

#### 2.8.3 潜在不一致风险

1. **deps 失败静默继续**
   - 位置：[block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L411)
   - 风险：`dbt deps` 失败时，依赖包未安装，但主任务继续执行，可能导致更隐蔽的错误
   - 建议：增加 deps 成功检查，或至少输出警告日志

2. **YAML Block 无输出变量**
   - 位置：[block_yaml.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_yaml.py#L92-L195)
   - 风险：YAML Block 执行后不存储任何输出变量，下游 block 无法消费其结果
   - 影响：dbt YAML Block 只能作为终端节点，不能作为数据流的中间节点

3. **show 命令与主任务共享 args**
   - 位置：[block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L424-L425)
   - 风险：show 命令复用了主任务的 args（包括 `--select`、`--vars` 等），并追加了 `--limit`
   - 注意：某些 dbt 命令的参数对 show 可能无效或有不同含义

4. **`--limit -1` 的语义**
   - 位置：[block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L402)
   - 待确认：dbt show 的 `--limit -1` 是否真的表示"不限行数"，还是会报错或有其他行为

#### 2.8.4 四条 Notebook 执行路径对比表

| 维度 | Preview (show) | Run | Test | Build |
|------|----------------|-----|------|-------|
| 触发按钮 | Preview (▶️) | Run | Test | Build |
| run_settings | `{}` | `{run_model:true}` | `{test_model:true}` | `{build_model:true}` |
| dbt 主任务 | 无（跳过） | run / snapshot | test | build |
| dbt show | ✅ 执行 | ✅ 执行 | ❌ 不执行 | ✅ 执行 |
| 模型物化 | ❌ 不物化 | ✅ 物化 | -（测试） | ✅ 物化 |
| df 输出 | ✅ 有 | ✅ 有 | ❌ 无 | ✅ 有 |
| 存储键 | `df` | `df` | `df` (None) | `df` |
| 数据库写入 | 只读 | 读写 | 只读（测试结果） | 读写 |
| 典型耗时 | 短（只查询） | 中 | 短 | 长 |

---

## 三、完整执行流程拆解

### 3.1 SQL Block 典型执行流（后台调度）

```
Pipeline Scheduler
  └─ BlockExecutor.execute_block(block_uuid)
       ├─ logger_manager 初始化
       ├─ block._execute_block()
       │    ├─ __create_upstream_tables()
       │    │    ├─ __upstream_blocks_from_sources()      # 从 SQL 中的 {{ source() }} 提取上游
       │    │    ├─ 获取上游 block 输出 (DataFrame)
       │    │    └─ DBTBlock.materialize_df()             # CSV → dbt seed → 物化表
       │    │         ├─ 写 CSV 到 seed-paths
       │    │         ├─ Profiles(project_path, variables) # 创建插值 profiles
       │    │         ├─ DBTCli.invoke(['seed', ...])      # dbt seed
       │    │         └─ 删除 CSV
       │    ├─ __task() → 'build'                         # 后台调度默认 build
       │    ├─ 构建 CLI 参数
       │    │    ├─ --project-dir {project_path}
       │    │    ├─ --select {model_name}
       │    │    ├─ --vars {json}
       │    │    └─ --target {target} (可选)
       │    ├─ Profiles(project_path, variables).__enter__()
       │    │    ├─ 读取 profiles.yml
       │    │    ├─ Jinja2 渲染变量
       │    │    └─ 写入临时目录
       │    ├─ DBTCli.invoke(['deps', ...])               # dbt deps
       │    ├─ DBTCli.invoke(['build', ...])              # dbt build (run + test)
       │    ├─ (如有下游非dbt block) DBTCli.invoke(['show', ...])  # 获取输出
       │    │    └─ cli.to_pandas(res)                    # agate → DataFrame
       │    ├─ Profiles.__exit__()                        # 清理临时 profiles
       │    └─ store_variables({'output_0': df})          # 存储输出
       └─ block.run_tests() → no-op                      # dbt 内部已处理测试
```

### 3.2 SQL Block Notebook 交互执行流（四条路径）

#### 3.2.1 Preview 路径（dbt show，仅查询）

**触发**：点击 Preview 按钮 或 Cmd/Ctrl + Enter
**run_settings**：`{}`（空字典）
**主任务**：show（不物化）

```
用户点击 Preview
  └─ WebSocket 消息（无 run_settings 字段）
       └─ output_display.py: run_settings or {} → {}
            └─ _execute_block(from_notebook=True, run_settings={})
                 ├─ __create_upstream_tables()  ← 物化上游（如有）
                 ├─ __task() → 'show'            ← run_settings={} → else 分支
                 ├─ 构建 CLI 参数
                 ├─ Profiles().__enter__()
                 ├─ dbt deps                       ← 始终执行
                 ├─ 主任务：跳过（task == 'show'）
                 ├─ needs_preview_df = True          ← from_notebook + task != 'test'
                 ├─ dbt show --limit 1000           ← 执行 show 获取预览
                 ├─ cli.to_pandas(res) → df
                 ├─ Profiles.__exit__()
                 ├─ store_variables({'df': df})   ← 存储为 df
                 └─ return [df]
```

**特点**：只查询不物化，速度快，用于验证 SQL 逻辑

---

#### 3.2.2 Run 路径（dbt run，物化模型）

**触发**：点击 Run 按钮
**run_settings**：`{run_model: true}`
**主任务**：run / snapshot

```
用户点击 Run
  └─ WebSocket 消息 { run_settings: { run_model: true } }
       └─ _execute_block(from_notebook=True, run_settings={run_model:true})
            ├─ __create_upstream_tables()
            ├─ __task() → 'run' (或 'snapshot')    ← run_model=true
            ├─ 构建 CLI 参数
            ├─ Profiles().__enter__()
            ├─ dbt deps
            ├─ dbt run --select ...                   ← 执行主任务（物化表）
            ├─ success? → 失败则抛出异常
            ├─ needs_preview_df = True
            ├─ dbt show --limit 1000                 ← 再执行 show 预览结果
            ├─ cli.to_pandas(res) → df
            ├─ Profiles.__exit__()
            ├─ store_variables({'df': df})
            └─ return [df]
```

**特点**：先物化模型 + 预览结果，执行两次 dbt 命令（run + show）

---

#### 3.2.3 Test 路径（dbt test，执行测试）

**触发**：点击 Test 按钮
**run_settings**：`{test_model: true}`
**主任务**：test

```
用户点击 Test
  └─ WebSocket 消息 { run_settings: { test_model: true } }
       └─ _execute_block(from_notebook=True, run_settings={test_model:true})
            ├─ __create_upstream_tables()
            ├─ __task() → 'test'                    ← test_model=true
            ├─ 构建 CLI 参数
            ├─ Profiles().__enter__()
            ├─ dbt deps
            ├─ dbt test --select ...                  ← 执行测试
            ├─ success? → 失败则抛出异常
            ├─ needs_preview_df = False                  ← task == 'test'
            ├─ 不执行 dbt show
            ├─ df = None
            ├─ Profiles.__exit__()
            ├─ store_variables({'df': None})          ← 存储 None
            └─ return [None]
```

**特点**：只执行测试，无数据输出，df 为 None

---

#### 3.2.4 Build 路径（dbt build，run + test）

**触发**：点击 Build 按钮
**run_settings**：`{build_model: true}`
**主任务**：build

```
用户点击 Build
  └─ WebSocket 消息 { run_settings: { build_model: true } }
       └─ _execute_block(from_notebook=True, run_settings={build_model:true})
            ├─ __create_upstream_tables()
            ├─ __task() → 'build'                    ← build_model=true
            ├─ 构建 CLI 参数
            ├─ Profiles().__enter__()
            ├─ dbt deps
            ├─ dbt build --select ...                 ← 执行 build (run + test)
            ├─ success? → 失败则抛出异常
            ├─ needs_preview_df = True
            ├─ dbt show --limit 1000                   ← 再执行 show 预览
            ├─ cli.to_pandas(res) → df
            ├─ Profiles.__exit__()
            ├─ store_variables({'df': df})
            └─ return [df]
```

**特点**：最完整的执行路径，build + show，三次 dbt 命令（deps + build + show）

### 3.3 YAML Block 执行流

```
BlockExecutor.execute_block()
  └─ DBTBlockYAML._execute_block()
       ├─ task = configuration.get('command', 'run')
       ├─ content = interpolate_content(block_content)    # Jinja2 插值
       ├─ args = shlex.split(content)                     # 解析用户命令
       ├─ 合并 --project-dir, --vars, --target, --profiles-dir
       ├─ Profiles().__enter__()
       ├─ DBTCli.invoke(['deps'] + args)
       ├─ DBTCli.invoke([task] + args)
       └─ Profiles().__exit__()
```

---

## 四、Preview 路径副作用边界深度分析

> **场景**：用户在 Notebook 中点 Preview 按钮或 Cmd+Enter 触发
> **入口条件**：`from_notebook=True` + `run_settings={}`（经 `run_settings or {}` 转换后）
> **任务选择**：`__task()` → `'show'`

### 4.1 副作用边界三件事的区分

Preview 路径的代码执行涉及三个独立操作空间，它们的副作用边界必须严格区分：

| 操作 | 执行阶段 | 对数据库的影响 | 对本地文件的影响 | 代码位置 |
|------|---------|--------------|----------------|----------|
| **① 上游 DataFrame seed 写库** | `_execute_block` 开头，`dbt deps` **之前** | ✅ **写入数据库**（创建/覆盖表） | ✅ 临时写 CSV → seed → 删除 | [block_sql.py#L332-L338](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L332-L338) |
| **② 当前模型是否物化** | 主任务阶段 | ❌ **不物化**（task='show' 跳过主任务） | - | [block_sql.py#L414-L418](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L414-L418) |
| **③ show 查询预览** | 最后阶段 | ❌ **只读查询**（`SELECT * FROM (compiled_sql) LIMIT n`） | - | [block_sql.py#L423-L430](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L423-L430) |

### 4.2 副作用一：上游 DataFrame seed 写库（Preview 也触发）

**关键结论：即使是 Preview，上游非 dbt block 的输出也会被写入数据库。**

#### 执行链路：

```
_execute_block() 第 332 行
  └─ __create_upstream_tables()                          ← 无条件调用，无任何 from_notebook 判断
       ├─ 对每个上游 source block:
       │    ├─ 获取 output（DataFrame）
       │    └─ 非空 → DBTBlock.materialize_df()          ← 每次都执行
       │         ├─ 生成 mage_{pipeline}_{block}.csv     ← 写本地临时文件
       │         ├─ dbt seed --select ... --full-refresh ← 写入数据库（DROP + CREATE + INSERT）
       │         └─ seed_path.unlink()                   ← 删除本地 CSV
```

**代码证据**（[block_sql.py#L332-L338](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L332-L338)）：

```python
# Create upstream tables
self.__create_upstream_tables(
    execution_partition=execution_partition,
    global_vars=global_vars,
    logger=logger,
    outputs_from_input_vars=outputs_from_input_vars,
    runtime_arguments=runtime_arguments,
)
```

> ⚠️ 注意 `__create_upstream_tables()` 调用 **没有任何条件判断**，不论 `task` 是 'show'/'run'/'test'/'build'，也不论 `from_notebook` 是 True/False，都无条件执行。

#### seed 命令的副作用（[block.py#L192-L201](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block.py#L192-L201)）：

```python
args = [
    'seed',
    '--project-dir', project_path,
    '--profiles-dir', profiles.profiles_dir,
    '--target', target,
    '--select', f'mage_{pipeline_uuid}_{block_uuid}',
    '--vars', cls._variables_json(merge_dict(variables, template_vars)),
    '--full-refresh'          # ← 全量刷新：DROP 旧表 → CREATE 新表 → 插入数据
]
DBTCli(logger=logger).invoke(args)   # ← 注意：此处也不检查 success！
seed_path.unlink()                   # ← 即使 seed 失败也会尝试删除 CSV
```

**副作用影响链**：
1. **数据库中会持久存在** `mage_{pipeline}_{block}` 表
2. 表数据是当前 Preview 时刻上游 block 的输出快照
3. **--full-refresh** 意味着每次 Preview 都会 **先删除再重建** 该表（不是追加）
4. 若有多个用户/进程同时 Preview，存在 **seed 竞争写库** 风险

#### seed 失败的异常行为：

- `DBTCli.invoke(seed_args)` 返回 `dbtRunnerResult`，但 **不检查 `res.success`**
- 即使 seed 失败，`seed_path.unlink()` 仍在同一缩进层级执行（`with Profiles` 块外）
- **但**：如果 dbt seed 内部抛出未捕获异常（而非通过 `res.success=False` 返回），则 Python 异常会向上传播，`seed_path.unlink()` 不会执行 → CSV 文件残留

### 4.3 副作用二：当前模型是否物化（Preview 不物化）

**关键结论：Preview 路径下当前 dbt 模型不会被物化到数据库。**

代码判断逻辑（[block_sql.py#L414-L418](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L414-L418)）：

```python
# run primary task, except for show
if task != 'show':          # ← Preview 时 task='show'，因此跳过
    res = cli.invoke([task] + args)
    success = res.success
    if not success:
        raise Exception(str(res.exception))
```

**完整对比矩阵**：

| 路径 | task 值 | 主任务执行？ | 当前模型物化？ |
|------|---------|-------------|--------------|
| **Preview** | `'show'` | ❌ 跳过 if 分支 | ❌ 不物化 |
| **Run** | `'run'`/`'snapshot'` | ✅ dbt run | ✅ 物化（CREATE TABLE AS SELECT...） |
| **Test** | `'test'` | ✅ dbt test | ❌ 测试断言不写库 |
| **Build** | `'build'` | ✅ dbt build | ✅ 物化（run + test 集成） |
| **后台调度** | `'build'` 或 `'run'` | ✅ 执行 | ✅ 物化 |

> 💡 Preview 的核心设计：**不修改当前模型的目标表，只通过 show 查询预览结果。**

### 4.4 副作用三：show 查询预览（只读）

**关键结论：show 是纯只读查询，不写入任何数据库对象。**

执行代码（[block_sql.py#L423-L430](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L423-L430)）：

```python
if needs_downstream_df or needs_preview_df:  # Preview: needs_preview_df=True
    args += (["--limit", str(limit)])        # limit 默认 1000 行
    res = cli.invoke(['show'] + args)        # ← dbt show 命令
    if res.success:
        df = cli.to_pandas(res)              # 转为 DataFrame
    else:
        raise Exception(str(res.exception))  # ← show 失败 → 整体失败
```

#### dbt show 的内部语义：

1. dbt 先编译模型 SQL（等同于 `dbt compile`）
2. 生成包装查询：`SELECT * FROM (编译后的模型SQL) LIMIT n`
3. 在目标数据库中 **执行该 SELECT 查询**
4. 返回查询结果的 `agate_table` 到内存

**副作用确认**：
- ✅ **不创建** 任何表/视图
- ✅ **不写入** 任何数据
- ✅ **不修改** dbt manifest/run_results（除非有特殊配置）
- ⚠️ 会占用数据库查询资源（计算资源、I/O、连接）
- ⚠️ limit 默认 1000 行，但大数据量模型的编译 + LIMIT 查询仍可能很重

#### Preview 路径的 needs_preview_df 条件：

```python
needs_preview_df = False
if from_notebook and task != 'test':   # Preview: from_notebook=True, task='show'!='test'
    needs_preview_df = True            # ← 满足
    limit = self.configuration.get('limit', 1000)
```

> Preview 路径必然满足 `needs_preview_df=True`，因此 show **一定会执行**。

### 4.5 Preview 完整副作用时间线

```
时间轴 →
│
├─ __create_upstream_tables()
│    ├─ [副作用] 上游 N 个 block → dbt seed --full-refresh
│    │    └─ 数据库：创建/覆盖 mage_{pipeline}_{block} 表  ← ✅ 写库！
│    └─ [副作用] 临时 CSV 文件（写→删）
│
├─ with Profiles():
│    ├─ [副作用] 创建临时 .profiles_interpolated_temp_xxx/ 目录
│    │
│    ├─ cli.invoke(['deps'] + args)   ← 始终执行，不检查
│    │
│    ├─ if task != 'show':  ← Preview 跳过此分支  ← ❌ 当前模型不物化！
│    │    └─ dbt run/build/test...
│    │
│    ├─ needs_preview_df = True
│    ├─ cli.invoke(['show', '--limit', '1000'] + args)  ← ✅ 只读查询
│    │    └─ 数据库：SELECT * FROM (model_sql) LIMIT 1000
│    │
│    └─ Profiles.clean()  ← 删除临时目录（__exit__ 触发）
│
├─ store_variables({'df': df})  ← 存储到内存变量系统
│
└─ return [df]
```

### 4.6 三件事的副作用隔离总结

| 操作 | Preview 是否触发 | 写库？ | 影响对象 | 失败时是否中断整体？ |
|------|----------------|-------|---------|-------------------|
| **上游 seed 写库** | ✅ 是 | ✅ 是 | 上游 source 表 | seed 返回失败不中断；抛异常才中断 |
| **当前模型物化** | ❌ 否（task='show'） | - | 模型目标表 | - |
| **show 查询预览** | ✅ 是 | ❌ 只读 | 无（查询结果仅在内存） | ✅ show 失败 → raise → 中断 |

> **重要警示**：用户点击 Preview 期望"只看看不修改"，但 **上游 block 的 seed 实际上正在写入数据库**。这是一个可能造成用户误解的副作用边界。

---

## 五、deps 调用失败与异常对主任务的影响分析

### 5.1 dbtRunner.invoke() 的两种失败模式

首先明确 dbtRunner 返回结果的两种失败模式（基于 dbtRunner 官方文档和社区验证）：

| 失败模式 | 触发条件 | Python 行为 | `res.success` | `res.exception` |
|---------|---------|------------|--------------|----------------|
| **模式 A：返回失败** | dbt 业务层面失败（如 SQL 语法错误、依赖包下载失败、测试不通过） | **不抛异常**，正常返回 `dbtRunnerResult` 对象 | `False` | 填充异常对象 |
| **模式 B：抛出异常** | dbt 运行时崩溃（如配置文件格式错误、未捕获的 Python 异常、`KeyboardInterrupt`、`SystemExit`） | **直接抛出 Python 异常**，不返回结果 | - | - |

**代码证据**（[dbt_cli.py#L85-L88](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_cli.py#L85-L88)）：

```python
res = dbt.invoke(cli_args)   # ← 仅在此处调用，无 try-except 包裹
self.result.append(res)
return res                   # ← 正常返回，若 dbt 抛异常则此行不执行
```

`DBTCli.invoke()` 完全不做异常捕获，直接将 dbtRunner 的行为向上透传。

---

### 5.2 deps 在 SQL Block 中的失败影响分析

**调用代码**（[block_sql.py#L411](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py#L411)）：

```python
cli.invoke(['deps'] + args)   # ← 无变量接收，无 try-except，不检查 success
```

#### 场景 1：deps 返回 success=False（模式 A，最常见）

**典型原因**：
- `packages.yml` 中引用的包版本不存在
- 网络不通导致无法下载 hub.getdbt.com 上的包
- git 引用的包仓库 URL 无效
- 本地包路径找不到

**执行结果**：

```
deps 执行 → res.success=False，不抛异常
    │
    ├─ ✅ deps 调用行本身不中断（没有 raise）
    ├─ → 继续执行后续所有步骤：
    │    ├─ if task != 'show': 主任务执行
    │    │    └─ 若主任务用到了缺失的宏/模型 → 主任务失败，抛 Exception
    │    ├─ if needs_*_df: dbt show 执行
    │    ├─ store_variables(...)
    │    └─ return [df]
    │
    └─ Profiles.__exit__() → clean() 正常执行
```

**最终用户体验**：
- 若主任务/show 恰好不依赖缺失包中的内容，**整个 block 可能成功完成，但依赖包实际上未正确安装**
- 若主任务需要缺失包中的宏/模型，主任务阶段才会失败，错误信息可能指向"找不到宏 XXX"而非"deps 安装失败"，增加排查难度

#### 场景 2：deps 内部抛出 Python 异常（模式 B，罕见但严重）

**典型原因**：
- `dbt_project.yml` 或 `packages.yml` YAML 格式完全错误（不是合法 YAML）
- dbt 内部模块导入失败（如 dbt 版本与适配器不兼容）
- 磁盘空间不足写缓存时报 IOError
- 用户中断（Ctrl+C）

**执行结果**：

```
deps 执行 → 抛出 Python 异常（如 YAMLError、ImportError、KeyboardInterrupt）
    │
    ├─ ❌ 异常从 cli.invoke() 向上冒泡
    ├─ → 跳过主任务、跳过 show
    ├─ → 不执行 store_variables
    ├─ → 不执行 return [df]
    │
    └─ 但 Profiles.__exit__(*args) 仍会被调用（with 语句保证）
         └─ Profiles.clean() 正常执行，临时目录被删除
```

**最终用户体验**：
- block 整体失败，异常堆栈从 `_execute_block` 向上传递
- 前端 Notebook 显示异常 traceback
- 但临时 profiles 目录被正确清理（with 语句保证）

---

### 5.3 deps 在 YAML Block 中的失败影响分析

**调用代码**（[block_yaml.py#L168-L174](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_yaml.py#L168-L174)）：

```python
cli.invoke(["deps"] + args)            # ← 同样不检查 success

res = cli.invoke(                      # ← 主任务
    args=[task] + args,
    log_level=log_level,
)
if not res.success:
    raise Exception(str(res.exception))
```

**行为与 SQL Block 完全一致**：
- ✅ **模式 A（success=False）**：deps 静默失败 → 主任务继续 → 主任务用到缺失内容时才失败
- ❌ **模式 B（抛异常）**：异常直接冒泡 → 主任务和 show 都跳过

额外注意：YAML Block 没有 show 步骤，因此主任务失败后直接终止，没有后续步骤。

---

### 5.4 deps 在 materialize_df() 中的 seed 命令失败分析

`materialize_df()` 中也有一个 `invoke()` 调用（seed 命令），其失败模式分析：

**调用代码**（[block.py#L201-L203](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block.py#L201-L203)）：

```python
DBTCli(logger=logger).invoke(args)   # ← 不检查 success，也不捕获异常

seed_path.unlink()                   # ← 上一行正常返回才执行
```

| 失败模式 | seed_path.unlink() 是否执行？ | CSV 文件是否残留？ | 是否影响后续 deps/主任务？ |
|---------|---------------------------|-----------------|------------------------|
| **模式 A**（success=False） | ✅ 执行（返回了 res 对象） | ❌ 正常删除 | ❌ 不影响（seed 返回了结果） |
| **模式 B**（抛异常） | ❌ 不执行（异常冒泡） | ✅ **残留磁盘** | ❌ 影响：异常直接向上冒泡 → 整个 block 失败 |

> ⚠️ **文件残留风险**：seed 抛异常时 CSV 文件留在 `seeds/mage_{pipeline}/` 目录下。如果是大的 DataFrame，可能占用较多磁盘空间。

---

### 5.5 主任务和结果反馈的完整影响矩阵

综合 SQL Block 中所有步骤（seed → deps → 主任务 → show），完整影响矩阵：

| 步骤 | 失败模式 | 是否中断？ | 后续步骤？ | store_variables？ | 临时目录清理？ |
|------|---------|----------|----------|-----------------|--------------|
| **seed（上游表）** | A: success=False | ❌ 不中断 | 继续（deps、主任务、show） | ✅ 执行 | ✅ 执行 |
| **seed（上游表）** | B: 抛异常 | ✅ 中断 | 全部跳过 | ❌ 不执行 | ✅ with 保证（仅 seed 内部的 Profiles） |
| **deps** | A: success=False | ❌ 不中断 | 继续（主任务、show） | ✅ 执行 | ✅ 执行 |
| **deps** | B: 抛异常 | ✅ 中断 | 主任务、show 跳过 | ❌ 不执行 | ✅ with 保证 |
| **主任务** | A: success=False | ✅ `raise Exception(...)` | show 跳过 | ❌ 不执行 | ✅ with 保证 |
| **主任务** | B: 抛异常 | ✅ 异常冒泡 | show 跳过 | ❌ 不执行 | ✅ with 保证 |
| **show** | A: success=False | ✅ `raise Exception(...)` | 无后续 | ❌ 不执行 | ✅ with 保证 |
| **show** | B: 抛异常 | ✅ 异常冒泡 | 无后续 | ❌ 不执行 | ✅ with 保证 |

---

### 5.6 deps 行为设计缺陷总结

**核心问题**：`dbt deps` 是 **唯一既不检查 success 也不捕获异常** 的关键前置步骤。

| 检查维度 | deps | 主任务 | show | seed |
|---------|------|-------|------|------|
| `if not res.success: raise` | ❌ 没有 | ✅ 有 | ✅ 有 | ❌ 没有 |
| 对整个 block 成功的语义必要性 | 高（依赖缺失通常导致主任务失败） | 极高 | 中/低（预览不应影响核心任务） | 高（上游数据缺失通常导致模型错误） |
| **一致性判定** | ❌ 缺失检查 | ✅ 正常 | ⚠️ 过于严格 | ❌ 缺失检查 |

**建议改进**：
1. **至少记录警告**：`if not res.success: logger.warn('dbt deps failed: ...')`
2. **deps 失败直接终止**（推荐）：主任务在依赖缺失时继续执行没有意义，只会产生更难排查的错误
3. **seed 也应检查 success**：上游表没有正确写入时，主任务的结果也不可信
4. **show 失败降级**：主任务成功但 show 失败时，不应该判定整个 block 失败（或至少可配置）

---

## 六、潜在风险分析

### 6.1 环境隔离风险（高危）

| 风险项 | 详情 |
|--------|------|
| **进程内执行无隔离** | dbt 在 Mage 主进程内通过 `dbtRunner` 执行，dbt 异常/内存泄漏可能影响 Mage 主进程稳定性 |
| **全局适配器注册表** | `reset_adapters()` 虽然在 `DBTAdapter.open()` 中调用，但 `DBTCli` 执行路径 **不调用** `reset_adapters()`，可能存在适配器状态残留 |
| **profiles 临时文件泄漏** | 虽然有 `__enter__`/`__exit__` 和 `__del__` 清理，但 Python 的 `__del__` 调用时机不确定，异常场景可能导致临时目录残留 |
| **dbt 版本兼容** | `dbtRunner` 接口在不同 dbt 版本间存在差异，Mage 代码中直接 `from dbt.cli.main import dbtRunner` 无版本约束 |

### 6.2 并发执行风险（中危）

| 风险项 | 详情 |
|--------|------|
| **dbt 全局状态** | dbt 的 `flags.set_from_args()` 设置全局标志，并发执行不同 block 时可能互相干扰 |
| **文件系统竞争** | 多个 block 同时写同一 dbt 项目的 `target/` 目录（compiled SQL、run_results.json 等），可能导致文件损坏 |
| **seed 物化竞争** | `materialize_df()` 写 CSV → seed → 删除 CSV 的过程不是原子操作，并发时可能互相干扰 |

### 6.3 错误处理风险（中危）

| 风险项 | 详情 |
|--------|------|
| **异常信息丢失** | `raise Exception(str(res.exception))` 将 dbt 异常转为字符串，丢失原始堆栈跟踪 |
| **deps 失败静默继续** | `cli.invoke(['deps'] + args)` 执行后**不检查 success**，依赖安装失败时主任务仍继续执行，可能导致更隐蔽的错误 |
| **静默失败** | `DBTAdapter.open()` 中异常被 `print()` 吞掉（非 debug 模式），返回 `None` 而非抛出异常 |
| **dbt list 失败降级** | `upstream_dbt_blocks()` 中 `dbt list` 失败时降级为只返回自身 block，可能导致依赖图不完整 |
| **Profiles 异步兼容** | `Profiles.profiles` 属性通过 `ThreadPoolExecutor` 处理异步上下文，但异常传播可能不完整 |
| **show 失败即整体失败** | 主任务成功但 show 失败时，整个 block 判定为失败——预览失败不应该影响核心任务的成功状态 |

### 6.4 安全风险（中危）

| 风险项 | 详情 |
|--------|------|
| **profiles 明文** | 插值后的 `profiles.yml` 以明文写入临时目录，包含数据库凭据，若清理失败则凭据残留磁盘 |
| **Jinja2 注入** | Profiles 插值使用 Jinja2 `Template(content).render(variables=..., **get_template_vars())`，`get_template_vars()` 的内容需要审查 |
| **变量序列化** | `_variables_json()` 序列化变量传递给 `--vars`，`encode_complex` 处理可能泄露敏感信息 |

### 6.5 功能完整性风险（低危）

| 风险项 | 详情 |
|--------|------|
| **SKIP_LIMIT_ADAPTER_NAMES** | SQL Server/Synapse/Fabric 适配器不支持 `--limit` 参数，代码中有常量定义但未在 `_execute_block` 中实际使用 |
| **dbt deps 无重试** | `cli.invoke(['deps'] + args)` 无重试机制，网络不稳定时可能导致整个 block 执行失败 |
| **上游 block 无输出时** | `__create_upstream_tables()` 在上游输出为空时仅打印日志继续执行，可能导致 dbt 模型引用不存在的 source |

---

## 七、待验证问题

### 7.1 环境隔离

1. **并发 block 执行时 `flags.set_from_args()` 是否线程安全？** dbt 的 flags 模块是否使用线程本地存储？
2. **`reset_adapters()` 是否在 `DBTCli` 路径中被调用？** 若否，连续执行不同数据库类型的 dbt block 时是否会出现适配器冲突？
3. **`dbtRunner` 是否支持并发 `invoke()` 调用？** 同一进程内多个线程同时调用 `dbtRunner.invoke()` 是否安全？

### 7.2 依赖管理

4. **`dbt deps` 的包安装目录是否在 `--project-dir` 下？** 多项目并发 `deps` 是否会冲突？
5. **`packages.yml` 中引用的本地包路径解析**，在 Mage 项目平台模式（platform project）下是否正确？
6. **`dbt list` 依赖图解析在大型项目中的性能**，数百个模型的 dbt 项目在 UI 中添加 block 时是否会超时？

### 7.3 数据流转

7. **`materialize_df()` 的 `--full-refresh` 策略**，在大数据量场景下是否会成为性能瓶颈？是否有增量物化方案？
8. **`dbt show --limit` 的实际 SQL 生成**，不同适配器（如 BigQuery、Snowflake）对 `LIMIT` 子句的支持是否存在差异？
9. **`to_pandas()` 方法** 对 `agate_table` 的转换，在 NULL 值、日期类型、嵌套结构等边界场景下是否正确？
10. **`--limit -1` 的语义**，dbt show 是否支持 -1 表示"不限行数"？还是会报错或有其他行为？

### 7.4 错误恢复

11. **dbt 执行超时机制**，`DBTCli.invoke()` 无超时参数，长时间运行的 dbt 模型是否会阻塞 Mage 进程？
12. **`dbt build` 部分失败时的行为**，当 build 中某个模型失败时，`dbtRunnerResult.success` 为 False，但已成功的模型如何回滚？
13. **`Profiles.clean()` 失败的影响**，临时目录清理失败是否会影响后续执行？
14. **deps 失败的设计意图**，`dbt deps` 不检查 success 是故意设计（容忍部分失败）还是遗漏？需要确认。
15. **show 失败的错误等级**，主任务成功但 show 失败时，是否应该判定为整个 block 失败？还是只记录警告？

### 7.5 调度集成

16. **`upstream_dbt_blocks()` 依赖图解析** 在 dbt 项目结构变更（添加/删除模型）后，Mage pipeline 是否自动更新依赖图？
17. **`mage_sources.yml` 的合并冲突**，多个 pipeline 共享同一 dbt 项目时，sources 文件是否会出现并发写入冲突？
18. **Pipeline 删除时的 source 清理**，删除 pipeline 后 `mage_sources.yml` 中的对应 source 是否被正确清理？

### 7.6 功能一致性

19. **YAML Block 是否应该支持 show 输出**，YAML Block 无 DataFrame 输出的设计是故意限制还是功能缺失？
20. **YAML Block 的返回值规范**，`_execute_block` 标注返回 `None`，但基类要求返回 `List`，是否违反接口契约？
21. **dbt deps 的缓存机制**，每次执行都跑 deps 是否必要？是否可以基于 `packages.yml` 变更做增量检测？
22. **disable_tests 配置的作用范围**，`disable_tests` 是仅影响后台调度，还是也影响 notebook 中的 Build 按钮？

---

## 八、核心文件索引

| 模块 | 文件路径 | 职责 |
|------|----------|------|
| **核心执行** | [dbt_cli.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_cli.py) | dbt 命令封装，dbtRunner 调用，日志回调 |
| | [dbt_adapter.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/dbt_adapter.py) | dbt 适配器连接管理，SQL 执行，Macro 执行 |
| | [block.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block.py) | DBTBlock 基类，DataFrame 物化，Sources 更新 |
| | [block_sql.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_sql.py) | SQL Block 执行逻辑，任务选择，依赖图解析 |
| | [block_yaml.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/block_yaml.py) | YAML Block 执行逻辑，自由命令解析 |
| **配置管理** | [profiles.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/profiles.py) | profiles.yml 插值与临时文件管理 |
| | [project.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/project.py) | dbt_project.yml 读取 |
| | [sources.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/sources.py) | mage_sources.yml 自动管理 |
| | [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/constants.py) | dbt block 常量定义 |
| | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/dbt/utils.py) | source 命名工具函数 |
| **缓存** | [cache.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/cache/dbt/cache.py) | dbt 项目缓存管理 |
| | [cache/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/cache/dbt/utils.py) | 缓存构建与文件扫描 |
| **dbt Cloud** | [dbt.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/services/dbt/dbt.py) | dbt Cloud API 客户端 |
| | [config.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/services/dbt/config.py) | dbt Cloud 配置 |
| | [services/constants.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/services/dbt/constants.py) | dbt Cloud 常量与状态枚举 |
| **调度集成** | [block_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/models/block/block_factory.py) | Block 类型工厂，DBT → DBTBlockSQL/DBTBlockYAML |
| | [block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/data_preparation/executors/block_executor.py) | Block 执行器，dbt 特殊处理逻辑 |
| | [configuration_option.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/settings/models/configuration_option.py) | 全局配置选项，dbt 项目/profile/target 发现 |
| **前端 & 通信** | [useCodeBlockProps.tsx](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/frontend/components/CodeBlockV2/dbt/useCodeBlockProps.tsx) | 前端 dbt block 按钮定义，run_settings 构造 |
| | [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/server/websocket_server.py) | WebSocket 服务器，接收执行请求 |
| | [output_display.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/server/utils/output_display.py) | 生成内核执行代码，run_settings or {} 转换 |
| | [execute_custom_code.py](file:///d:/fz/0601/solo-dogfeeding/code/312-mage-ai/mage_ai/server/utils/execute_custom_code.py) | 内核侧执行入口，调用 block.execute_with_callback |

---

## 九、总结

Mage AI 与 dbt 的集成实现了一条 **"配置插值 → 进程内命令执行 → 结果反馈"** 的完整链路，核心设计特点包括：

1. **双 Block 类型**：`DBTBlockSQL`（单模型精确控制）和 `DBTBlockYAML`（自由命令编排），覆盖不同使用场景
2. **Profiles 插值隔离**：通过临时目录 + Jinja2 渲染实现变量注入与环境隔离，但清理机制存在可靠性风险
3. **DataFrame ↔ dbt 双向桥接**：入站通过 `materialize_df()` + seed 物化，出站通过 `dbt show` + DataFrame 输出
4. **依赖图双向映射**：`upstream_dbt_blocks()` 将 dbt DAG 映射为 Mage pipeline DAG，但仅在 block 创建时执行
5. **无进程级隔离**：所有 dbt 执行在 Mage 主进程内完成，简化部署但牺牲了隔离性和稳定性

### 执行路径核心发现

- **四条 Notebook 路径**：Preview(show) / Run / Test / Build，由 `run_settings` 控制，经 `run_settings or {}` 转换后进入 `__task()` 方法
- **deps 无条件执行**：每次 block 执行必先跑 `dbt deps`，但不检查成功与否（潜在设计问题）
- **show 双触发条件**：`needs_preview_df`（Notebook 预览）和 `needs_downstream_df`（下游非 dbt block），任一满足即执行 show
- **输出键名分化**：Notebook 模式存 `df`，后台调度存 `output_0`，适配不同消费场景

### Preview 副作用边界核心发现

- **① 上游 seed 写库：Preview 也触发** → `__create_upstream_tables()` 在 `_execute_block` 开头无条件执行，即使 Preview 也会用 `dbt seed --full-refresh` 将上游 DataFrame 写入数据库
- **② 当前模型物化：Preview 不触发** → `if task != 'show'` 判断跳过主任务分支，当前模型不会被 CREATE/ALTER
- **③ show 查询预览：纯只读** → `SELECT * FROM (compiled_sql) LIMIT n` 包装查询，不写任何数据库对象
- **⚠️ 用户预期偏差**：用户以为 Preview"只看看"，但上游表的 seed 副作用实际上正在修改数据库

### deps 失败影响与错误处理核心发现

- **两种失败模式本质不同**：dbtRunner 的 `success=False`（业务失败，不抛异常）vs 抛出异常（运行时崩溃），两者的后续执行路径完全不同
- **deps/seed 缺失 success 检查**：`dbt deps` 和 `materialize_df()` 中的 seed 命令是仅有的两个不检查 `res.success` 的关键步骤，失败会静默继续
- **deps 模式 A 失败最隐蔽**：`success=False` 不中断 → 主任务继续 → 用到缺失内容时才报错，错误信息脱离上下文（"找不到宏" vs "deps 安装失败"）
- **deps 模式 B 失败但清理可靠**：抛出异常会跳过主任务，但 `with Profiles()` 语句保证 `__exit__` 仍会执行 → 临时 profiles 目录可靠清理
- **seed 抛异常有文件残留风险**：`seed_path.unlink()` 在 `invoke` 之后且不在 try-finally 中，seed 抛异常时 CSV 留在磁盘

### 一致性结论

- ✅ **错误处理一致**：SQL Block 和 YAML Block 都使用 `res.success` 判断 + `Exception(str(res.exception))` 抛出
- ✅ **deps 行为一致**：两者都不检查 deps 成功与否
- ❌ **数据输出不一致**：SQL Block 有完整的 show + DataFrame 输出链路，YAML Block 没有，导致 YAML Block 只能作为终端节点
- ❌ **返回值规范不一致**：YAML Block 的 `_execute_block` 标注返回 `None`，与基类 `List` 返回类型不符
- ⚠️ **错误检查强度不一致**：主任务/show 严格检查 success，deps/seed 完全不检查，show 的检查强度甚至高于语义上更重要的 deps

最主要的改进方向在于：**环境隔离**（进程/容器级隔离）、**并发安全**（全局状态保护）、**错误恢复**（超时与部分失败处理）、**一致性补齐**（YAML Block 输出能力 + deps/seed success 检查）和**安全加固**（凭据保护）。

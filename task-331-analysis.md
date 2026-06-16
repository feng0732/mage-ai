# Sensor 等待器代码分析

## 一、核心概念

Sensor（传感器）是 Mage AI 中的一种特殊块类型，用于**轮询等待某个条件满足**后才继续执行下游块。典型应用场景包括：
- 等待上游管道运行完成
- 等待某个数据文件出现
- 等待外部系统状态达到预期

Sensor 块的核心行为是：**循环执行条件检查 → 不满足则休眠 → 再检查 → 直到条件满足或超时**。

---

## 二、类结构与继承关系

### 2.1 类层次

```
Block (基类)
  └── SensorBlock (传感器块)
```

- [Block](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L349-L450)：所有块类型的基类，包含通用的执行、状态管理、变量存储等逻辑
- [SensorBlock](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L4274-L4305)：继承自 Block，重写了 `execute_block_function` 方法，实现轮询等待逻辑

### 2.2 关键属性

| 属性 | 定义位置 | 说明 |
|------|----------|------|
| `type` | [Block.__init__](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L383) | 块类型，Sensor 为 `BlockType.SENSOR` |
| `status` | [Block.__init__](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L389) | 块执行状态 |
| `timeout` | [Block.__init__](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L396) | 超时设置（但当前代码中 SensorBlock 未实际使用） |
| `upstream_blocks` | [Block.__init__](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L404) | 上游块列表 |
| `downstream_blocks` | [Block.__init__](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L405) | 下游块列表 |

---

## 三、前后依赖关系

### 3.1 块级依赖

Sensor 块通过 `upstream_blocks` 和 `downstream_blocks` 维护依赖关系：

- **上游依赖**：Sensor 块可以有上游块，只有所有上游块执行完成后，Sensor 才会开始执行
- **下游依赖**：Sensor 块条件满足后，下游块才会开始执行

依赖检查发生在 [run_blocks](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L246-L252) 和 [run_blocks_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L304-L311) 函数中：

```python
# 遍历所有上游块，检查是否都已完成
for upstream_block in block.upstream_blocks:
    if tasks.get(upstream_block.uuid) is None:
        blocks.put(block)  # 重新放回队列
        skip = True
        break
```

### 3.2 管道级依赖（跨管道 Sensor）

Sensor 的典型用法是**跨管道等待**，通过 [check_status](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/orchestration/run_status_checker.py#L24-L64) 函数实现：

```
管道 A (被等待)
    ↓
PipelineRun (运行记录)
    ↓
BlockRun (块运行记录)
    ↓
Sensor 块 (管道 B 中) → 查询数据库 → 判断条件是否满足
    ↓
管道 B 的下游块
```

**关键依赖关系**：
1. Sensor 函数通过 `pipeline_uuid` 和 `block_uuid` 参数指定要等待的目标
2. 从数据库查询 `PipelineRun` 和 `BlockRun` 记录
3. 根据 `execution_date` 在时间窗口内查找最近的运行记录
4. 根据运行状态返回 `True`（条件满足）或 `False`（继续等待）

---

## 四、状态变化流程

### 4.1 Block 状态（块级）

定义在 [BlockStatus](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/constants.py#L48-L52)：

| 状态 | 说明 |
|------|------|
| `NOT_EXECUTED` | 未执行，初始状态 |
| `EXECUTED` | 已执行成功 |
| `FAILED` | 执行失败 |
| `UPDATED` | 已更新（代码变更后） |

### 4.2 PipelineRun / BlockRun 状态（运行级）

定义在 [PipelineRunStatus](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/orchestration/db/models/schedules.py#L769-L774) 和 [BlockRunStatus](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1664-L1672)：

**PipelineRun 状态**：
| 状态 | 说明 |
|------|------|
| `INITIAL` | 初始化 |
| `RUNNING` | 运行中 |
| `COMPLETED` | 已完成 |
| `FAILED` | 失败 |
| `CANCELLED` | 已取消 |

**BlockRun 状态**：
| 状态 | 说明 |
|------|------|
| `INITIAL` | 初始化 |
| `QUEUED` | 已排队 |
| `RUNNING` | 运行中 |
| `COMPLETED` | 已完成 |
| `FAILED` | 失败 |
| `CANCELLED` | 已取消 |
| `UPSTREAM_FAILED` | 上游失败 |
| `CONDITION_FAILED` | 条件不满足 |

### 4.3 Sensor 执行的状态流转

```
初始状态: status = NOT_EXECUTED
    ↓
开始执行 (execute_sync)
    ↓
状态变为: 执行中 (隐式，无显式 RUNNING 状态)
    ↓
循环:
  ├─ 调用 sensor 函数检查条件
  ├─ 条件满足 → break 循环
  └─ 条件不满足 → sleep(60秒) → 继续循环
    ↓
执行成功 → status = EXECUTED
    ↓
或
执行失败 → status = FAILED
```

状态更新代码在 [execute_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1663-L1667)：

```python
if update_status:
    self.status = BlockStatus.EXECUTED
except Exception as err:
    if update_status:
        self.status = BlockStatus.FAILED
```

---

## 五、执行调用链

### 5.1 异步执行路径

```
run_blocks() [L170]
    ↓
create_block_task() → asyncio.create_task()
    ↓
Block.execute() [L1696]
    ↓
loop.run_in_executor() → 线程池执行
    ↓
Block.execute_sync() [L1436]
    ↓
__execute() 内部函数
    ↓
Block.execute_block() [L1855]
    ↓
Block._execute_block() [L1966]
    ↓
exec(block内容) → 收集 @sensor 装饰的函数
    ↓
Block._validate_execution() → 验证函数签名
    ↓
SensorBlock.execute_block_function() [L4275]  ← 重写的方法
    ↓
循环: block_function() → time.sleep(60)
    ↓
返回 [] (空列表)
```

### 5.2 同步执行路径

```
run_blocks_sync() [L267]
    ↓
Block.execute_sync() [L1436]
    ↓
(后续与异步路径相同)
```

### 5.3 关键方法详解

#### SensorBlock.execute_block_function

[SensorBlock.execute_block_function](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L4275-L4305) 是 Sensor 等待逻辑的核心：

```python
def execute_block_function(self, block_function, input_vars, from_notebook=False, global_vars=None, ...):
    if from_notebook:
        # Notebook 模式下不轮询，直接执行一次
        return super().execute_block_function(...)
    else:
        # 管道执行模式下进入轮询循环
        sig = signature(block_function)
        has_args = any([p.kind == p.VAR_POSITIONAL for p in sig.parameters.values()])
        has_kwargs = any([p.kind == p.VAR_KEYWORD for p in sig.parameters.values()])
        use_global_vars = has_kwargs and global_vars is not None and len(global_vars) != 0
        args = input_vars if has_args else []
        
        while True:
            # 执行条件检查函数
            condition = (
                block_function(*args, **global_vars) if use_global_vars else block_function()
            )
            if condition:
                break  # 条件满足，退出循环
            print('Sensor sleeping for 1 minute...')
            time.sleep(60)  # 休眠 60 秒
        
        return []  # Sensor 块不产生输出数据
```

**关键点**：
1. **两种模式**：`from_notebook=True` 时只执行一次（用于调试），否则进入轮询
2. **轮询间隔**：固定 60 秒（硬编码）
3. **返回值**：空列表 `[]`，Sensor 不产生数据输出
4. **参数传递**：支持 `*args` 和 `**kwargs` 两种形式传递输入变量

#### run_sensors 参数控制

Sensor 的执行受 `run_sensors` 参数控制，在 [run_blocks](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L230-L235) 中：

```python
if (
    block.type in NON_PIPELINE_EXECUTABLE_BLOCK_TYPES
    or not run_sensors
    and block.type == BlockType.SENSOR
):
    continue  # 跳过 Sensor 块
```

当 `run_sensors=False` 时，Sensor 块会被跳过不执行。

---

## 六、资源收放机制

### 6.1 变量存储

Sensor 块的输出是空列表 `[]`，但仍会经过变量存储流程：

1. [execute_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1592-L1640) 中处理输出：
   ```python
   block_output = self.post_process_output(output)
   variable_keys = [f'output_{idx}' for idx in range(output_count)]
   variable_mapping = dict(zip(variable_keys, block_output))
   ```

2. 调用 [store_variables](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3888) 存储变量

3. Sensor 返回 `[]`，所以 `output_count = 0`，实际不存储任何输出变量

### 6.2 内存管理

当 `MEMORY_MANAGER_V2` 启用时，通过 `execute_with_memory_tracking` 跟踪内存使用：

```python
output, self.resource_usage = execute_with_memory_tracking(
    block_function_updated,
    args=input_vars,
    kwargs=global_vars if has_kwargs and global_vars else None,
    ...
)
```

但在 SensorBlock 中，由于重写了 `execute_block_function`，内存跟踪**不会被应用**到轮询循环上（只有基类的 `execute_block_function` 使用了内存跟踪）。

### 6.3 资源释放

Sensor 块执行完成后，主要的资源清理包括：

1. **输出缓存重置**：[execute_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1648) 中重置 `self._outputs = None`

2. **变量删除策略**：当 `write_policy` 为 `FAIL` 或 `APPEND` 时，[delete_variables](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3724-L3758) 会有不同行为：
   - `FAIL`：如果变量已存在，抛出异常
   - `APPEND`：不删除现有变量，直接追加

3. **标准流重定向**：通过 [_redirect_streams](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1933-L1964) 上下文管理器管理 stdout/stderr，执行完毕后自动恢复

### 6.4 注意：资源泄漏风险

当前 SensorBlock 实现存在以下潜在问题：

1. **无超时机制**：`while True` 循环没有超时退出，如果条件永远不满足，会无限休眠下去
2. **无取消机制**：轮询过程中无法响应取消信号
3. **固定轮询间隔**：硬编码 60 秒，无法配置
4. **内存跟踪缺失**：重写的 `execute_block_function` 没有使用内存跟踪

---

## 七、Sensor 模板与典型用法

### 7.1 默认模板

[default.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/templates/sensors/default.py) 展示了典型用法：

```python
from mage_ai.orchestration.run_status_checker import check_status

@sensor
def check_condition(*args, **kwargs) -> bool:
    """
    Template code for checking if block or pipeline run completed.
    """
    return check_status(
        'pipeline_uuid',
        kwargs['execution_date'],
        block_uuid='block_uuid',  # 可选：等待特定块
        hours=24,  # 可选：时间窗口，默认 24 小时
    )
```

### 7.2 check_status 函数逻辑

[check_status](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/orchestration/run_status_checker.py#L24-L64) 的工作流程：

1. 验证管道和块是否存在
2. 根据 `execution_date` 和 `hours` 构建时间窗口
3. 查询数据库中最近的 `PipelineRun` 记录
4. 如果指定了 `block_uuid`，进一步查找对应的 `BlockRun`
5. 判断状态：
   - **失败状态**（FAILED / CANCELLED）：抛出异常，终止 Sensor
   - **完成状态**（COMPLETED）：返回 `True`，条件满足
   - **其他状态**（INITIAL / RUNNING 等）：返回 `False`，继续等待

---

## 八、完整执行流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     Pipeline 执行                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    run_blocks() / run_blocks_sync()
                              │
                              ▼
                    检查所有 upstream_blocks
                    是否已执行完成
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
                未完成                已完成
                    │                   │
                    │                   ▼
                    │         Block.execute() / execute_sync()
                    │                   │
                    │                   ▼
                    │         _redirect_streams (上下文)
                    │                   │
                    │                   ▼
                    │         execute_block()
                    │                   │
                    │                   ▼
                    │         _execute_block()
                    │                   │
                    │                   ▼
                    │         exec(block.content) → 收集 @sensor 函数
                    │                   │
                    │                   ▼
                    │         _validate_execution()
                    │                   │
                    │                   ▼
                    │         SensorBlock.execute_block_function()
                    │                   │
                    │                   ▼
                    │         ┌── while True: ──┐
                    │         │                 │
                    │         ▼                 │
                    │     调用 sensor 函数      │
                    │         │                 │
                    │    ┌────┴────┐            │
                    │    │         │            │
                    │    ▼         ▼            │
                    │  条件满足  条件不满足      │
                    │    │         │            │
                    │    │         ▼            │
                    │    │     sleep(60s)       │
                    │    │         │            │
                    │    │         └─────────────┘
                    │    │
                    │    ▼
                    └────┘
                         │
                         ▼
                 存储变量 (空输出)
                         │
                         ▼
                 status = EXECUTED
                         │
                         ▼
                 触发 downstream_blocks
```

---

## 九、关键代码文件索引

| 文件 | 位置 | 说明 |
|------|------|------|
| 块基类与 SensorBlock | [__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py) | Block 类、SensorBlock 类、执行流程 |
| 块常量定义 | [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/constants.py) | BlockType、BlockStatus 等枚举 |
| Sensor 模板 | [default.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/templates/sensors/default.py) | 默认 Sensor 模板代码 |
| 状态检查器 | [run_status_checker.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/orchestration/run_status_checker.py) | 跨管道状态检查函数 |
| 调度模型 | [schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/orchestration/db/models/schedules.py) | PipelineRun、BlockRun 模型 |
| 管道执行器 | [pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/executors/pipeline_executor.py) | 管道执行器 |
| 装饰器定义 | [decorators.py](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/decorators.py) | @sensor 等装饰器（桩定义） |

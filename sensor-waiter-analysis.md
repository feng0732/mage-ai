# Sensor 等待器：同步/异步开关差异与资源收放边界分析

---

## 一、同步 vs 异步执行路径：开关差异全表

Sensor 的运行行为受到三层开关的逐级传递影响：**管道层** → **块调度层** → **块执行层**。
同一参数在不同路径中取值来源不同，导致行为差异。

### 1.1 三层开关传递链

```
Pipeline.execute() / Pipeline.execute_sync()
    │
    │  run_sensors=True/False, parallel=True/False
    ▼
run_blocks() / run_blocks_sync()
    │
    │  run_sensors → from_notebook = not run_sensors
    │  parallel → 决定 execute() 内是否用线程池
    ▼
Block.execute() / Block.execute_sync()
    │
    │  from_notebook → 传入 execute_sync → 传入 execute_block
    ▼
SensorBlock.execute_block_function()
    │
    │  from_notebook → 决定轮询 or 单次执行
    ▼
```

### 1.2 各参数在各层的行为

| 参数 | 异步路径 `run_blocks` | 同步路径 `run_blocks_sync` | 差异说明 |
|------|----------------------|---------------------------|----------|
| **`run_sensors`** | [L177](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L177) 入参，用于**跳过** Sensor 块 [L230-235](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L230-L235) | [L273](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L273) 入参，但**不用于跳过** Sensor 块 [L289-293](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L289-L293) | **关键差异**：异步路径在调度层就跳过 Sensor；同步路径不跳过，而是在执行层通过 `from_notebook` 控制 |
| **`from_notebook`** | [L174](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L174) 入参，用于添加 logger | 不接收此参数，但内部用 `not run_sensors` 作为 `from_notebook` 传给 `execute_sync` [L328](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L328) | 同步路径中 `from_notebook` 由 `run_sensors` 反转得到 |
| **`parallel`** | [L176](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L176) 入参，传递到 `Block.execute()` [L211](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L211)，决定是否用 `run_in_executor` | 无此参数 | 仅异步路径存在，影响 `execute_sync` 的调用方式（直接调用 vs 线程池） |
| **Sensor 跳过逻辑** | `not run_sensors and block.type == BlockType.SENSOR` → `continue` [L232-233](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L232-L233) | 无此跳过逻辑，Sensor 总会被调度执行 | **最核心差异** |

### 1.3 `run_sensors=False` 时的行为对比

**异步路径** (`run_blocks`)：
```
run_sensors=False
    ↓
调度层：Sensor 块被 continue 跳过，不进入任务队列 [L232-235]
    ↓
Sensor 的 downstream_blocks 也不会被添加到队列
    ↓
结果：Sensor 及其下游完全不会执行
```

**同步路径** (`run_blocks_sync`)：
```
run_sensors=False
    ↓
调度层：Sensor 块不被跳过，正常进入执行
    ↓
execute_sync(from_notebook=not run_sensors = True) [L328]
    ↓
execute_block(from_notebook=True)
    ↓
SensorBlock.execute_block_function(from_notebook=True) [L4284]
    ↓
调用 super().execute_block_function() → 单次执行，不轮询
    ↓
结果：Sensor 只执行一次条件检查就返回
```

**影响**：在同步路径中 `run_sensors=False` 并不会跳过 Sensor，而是把它的行为从"轮询等待"降级为"单次执行"。Sensor 仍然会被标记为 `EXECUTED`，下游块照常执行。如果此时条件不满足，Sensor 的逻辑语义已被改变，但下游块不会感知到。

### 1.4 `from_notebook` 到 SensorBlock 的传递路径

`from_notebook` 在不同层级有不同的语义来源：

| 层级 | 异步路径 | 同步路径 |
|------|---------|---------|
| `run_blocks` / `run_blocks_sync` 入参 | 来自管道调用者 | 不接收此参数 |
| `Block.execute()` | `from_notebook = not run_sensors` [L1716](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1716) / [L1727](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1727) | 不经过 `execute()`，直接调 `execute_sync` |
| `Block.execute_sync()` | 来自 `execute()` 传入的 `from_notebook` | 来自 `not run_sensors` [L328](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L328) |
| `SensorBlock.execute_block_function()` | `from_notebook=True` → 单次执行；`False` → 轮询 [L4284](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L4284) | 同 |

**关键发现**：`from_notebook` 在执行层的实际含义是**"是否跳过轮询"**，而非字面上的"是否来自 Notebook"。参数名 `from_notebook` 具有误导性——当 `run_sensors=True` 时 `from_notebook=False`，Sensor 才会进入轮询；当 `run_sensors=False` 时 `from_notebook=True`，Sensor 只执行一次。

### 1.5 Pipeline.execute vs Pipeline.execute_sync 的根块选择差异

| 维度 | Pipeline.execute (异步) | Pipeline.execute_sync (同步) |
|------|------------------------|------------------------------|
| 根块过滤 | `len(block.upstream_blocks) == 0` [L772](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/pipeline.py#L772) | `upstream_blocks == 0 AND type in [DATA_EXPORTER, DATA_LOADER, DBT, TRANSFORMER, SENSOR]` [L817-823](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/pipeline.py#L817-L823) |
| SENSOR 是否可作为根块 | 是（任何无上游的块均可） | 是（显式列入白名单） |
| CALLBACK / CONDITIONAL 是否可作为根块 | 是（无上游即可） | **否**（不在白名单中） |

同步路径用白名单过滤根块，这意味着 Addon 类型（Callback / Conditional）不会被当作根块调度。Sensor 在两种路径下都可作为根块。

---

## 二、变量清理

### 2.1 Sensor 空输出对变量清理的影响

SensorBlock.execute_block_function 返回 `[]`（空列表），这会沿以下路径影响变量清理：

```
SensorBlock.execute_block_function() → 返回 []
    ↓
_execute_block() → outputs = [] → return []
    ↓
execute_block() → return dict(output=[])
    ↓
execute_sync.__execute() →
    output = dict(output=[])          # 来自 execute_block
    block_output = self.post_process_output(output)  # [] 或 [] 的 or
    output_count = len(block_output)  # = 0
    variable_keys = []                # range(0) 为空
    variable_mapping = {}             # dict(zip([], [])) = {}
```

[post_process_output](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1693-L1694) 的实现：
```python
def post_process_output(self, output: Dict) -> List:
    return output['output'] or []
```

**注意**：`[] or []` 的结果是 `[]`（空列表的布尔值为 False），所以 `output_count = 0`，`variable_mapping = {}`。

### 2.2 store_variables 对空 mapping 的处理

[store_variables](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3888) 接收到 `variable_mapping = {}` 时：

1. `__store_variables_prepare` 被调用 → `__consolidate_variables({})` → `output_variables = {}`, `print_variables = {}`
2. `removed_variables = []`（无旧变量需要移除）
3. 返回 `dict(removed_variables=[], variable_mapping={})`
4. `variables_data['variable_mapping']` 为空字典 → for 循环不执行 → 无变量写入
5. `skip_delete` 默认为 None → `delete_variables` 不被显式调用

**结果**：Sensor 执行完成后，不写入任何新变量，也不删除旧变量。如果 Sensor 块之前有遗留的变量文件，它们**不会被清理**。

### 2.3 delete_variables 的写入策略分支

[delete_variables](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3724-L3758) 的行为取决于 `write_policy`：

| write_policy | variable_object.data_exists() | 行为 |
|---|---|---|
| 无 (None) | — | 直接 `variable_object.delete()` [L3758](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3758) |
| FAIL | True | 抛出异常 |
| FAIL | False | 继续到 `variable_object.delete()` |
| APPEND | True | **直接 return，不删除** [L3755-3756](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3755-L3756) |
| APPEND | False | 继续到 `variable_object.delete()` |

**Sensor 特殊性**：Sensor 返回空列表，正常流程中 `delete_variables` 不会被 `store_variables` 显式调用。但如果启用 `MEMORY_MANAGER_V2` 且 Sensor 走基类的 `execute_block_function`（即 `from_notebook=True` 模式），基类中的 generator 处理逻辑会调用 `self.delete_variables()`。

### 2.4 变量清理遗漏场景

以下场景中变量**不会被清理**：

1. **Sensor 轮询模式**（`from_notebook=False`）：`store_variables` 收到空 mapping，不触发任何删除
2. **Sensor 之前的执行残留**：如果同一 Sensor 块历史执行中产生了变量（如 Notebook 模式），切换到管道执行后这些变量不会被清理
3. **APPEND 写策略下的重复执行**：`delete_variables` 遇到 APPEND 策略直接返回，即使正常块也不会删除变量

---

## 三、输出分析

### 3.1 analyze_outputs 对空映射的处理

[analyze_outputs](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3397-L3431) 接收到 `variable_mapping = {}` 时：

```python
def analyze_outputs(self, variable_mapping, execution_partition=None, shape_only=False):
    if self.pipeline is None:
        return
    for uuid, data in variable_mapping.items():  # 空字典 → 不进入循环
        ...
```

**结果**：空映射导致 for 循环不执行，`analyze_outputs` 什么都不做。这在 Sensor 场景下是正确的——没有输出数据就不需要分析。

### 3.2 两种 analyze_outputs 调用模式的差异

[execute_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1650-L1661) 中：

```python
if BlockType.CHART != self.type:
    if analyze_outputs:
        self.analyze_outputs(variable_mapping, execution_partition=execution_partition)
    else:
        self.analyze_outputs(variable_mapping, execution_partition=execution_partition, shape_only=True)
```

| 调用场景 | `analyze_outputs` 参数 | `shape_only` | 实际效果 |
|---------|----------------------|-------------|---------|
| 管道调度（`analyze_outputs=False`） | False | True | 只记录 shape 信息 |
| Notebook 调试（`analyze_outputs=True`） | True | False | 完整分析（统计、洞察等） |

**Sensor 场景**：无论哪种模式，`variable_mapping = {}` 都导致 `analyze_outputs` 不产生任何效果。但注意——**调用本身仍然发生了**，只是空输入导致空操作。

### 3.3 aggregate_summary_info 的边界条件

[aggregate_summary_info](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3760-L3774) 在 `execute_sync` 中被调用：

```python
if not is_dynamic_block_child(self):
    self.aggregate_summary_info()
```

此方法内部检查：
```python
def aggregate_summary_info(self, execution_partition=None):
    if not VARIABLE_DATA_OUTPUT_META_CACHE or not self.variable_manager:
        return
    aggregate_summary_info_for_all_variables(...)
```

**两个退出条件**：
1. `VARIABLE_DATA_OUTPUT_META_CACHE` 为 False（配置开关关闭）
2. `self.variable_manager` 为 None（管道未设置）

**Sensor 影响**：即使 Sensor 没有输出变量，`aggregate_summary_info_for_all_variables` 仍然会扫描磁盘上的变量目录。如果之前有残留变量文件，它们会被统计到汇总信息中，造成**过时的汇总数据**。

### 3.4 _outputs 缓存重置

```python
self._outputs = None  # [L1648]
```

无论输出是否为空，缓存都会被清除。这是正确的清理行为——确保下次访问 `self.outputs` 时会重新加载。但对于 Sensor，因为变量目录为空，重新加载也只会得到空结果。

---

## 四、内存记录边界

### 4.1 基类 execute_block_function 的内存跟踪

[Block.execute_block_function](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2152-L2167) 在 `MEMORY_MANAGER_V2=True` 时：

```python
if MEMORY_MANAGER_V2:
    output, self.resource_usage = execute_with_memory_tracking(
        block_function_updated,
        args=input_vars,
        kwargs=global_vars if has_kwargs and ... else None,
        logger=logger,
        logging_tags=logging_tags,
        log_message_prefix=log_message_prefix,
    )
```

内存跟踪覆盖了：
- 函数执行前的内存快照
- 函数执行后的内存快照
- 计算增量并记录到 `self.resource_usage`

### 4.2 SensorBlock 重写后的内存跟踪丢失

[SensorBlock.execute_block_function](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L4274-L4305) 完全重写了基类方法：

```python
class SensorBlock(Block):
    def execute_block_function(self, block_function, input_vars, from_notebook=False, global_vars=None, ...):
        if from_notebook:
            return super().execute_block_function(...)  # ← 走基类，有内存跟踪
        else:
            # ← 轮询路径，直接调用 block_function()，无内存跟踪
            while True:
                condition = block_function(*args, **global_vars) if use_global_vars else block_function()
                if condition:
                    break
                time.sleep(60)
            return []
```

**两条路径的差异**：

| 维度 | `from_notebook=True`（走 super） | `from_notebook=False`（轮询） |
|------|--------------------------------|-------------------------------|
| 内存跟踪 | ✅ `execute_with_memory_tracking` | ❌ 完全缺失 |
| `self.resource_usage` | 被设置为 `ResourceUsage` 对象 | 停留在初始化值 `None` |
| Generator 变量流式存储 | ✅ 逐条存储 + 类型汇总 | ❌ 不适用（返回 `[]`） |
| `delete_variables` 调用 | ✅ generator 模式下调用 | ❌ 不调用 |

### 4.3 execute_sync 层面的 MemoryManager

[execute_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1675-L1691) 中有一段被注释掉的 `MemoryManager` 上下文：

```python
# if MEMORY_MANAGER_V2:
#     with MemoryManager(
#         scope_uuid=...,
#         process_uuid='block.execute_sync',
#         ...
#     ):
#         return __execute()
return __execute()
```

**现状**：无论 `MEMORY_MANAGER_V2` 是否开启，`execute_sync` 层面都不再有外层内存管理。所有内存跟踪都依赖 `execute_block_function` 内部的 `execute_with_memory_tracking`。

**Sensor 影响**：Sensor 轮询路径既没有 `execute_sync` 层面的 `MemoryManager`（已注释），也没有 `execute_block_function` 层面的 `execute_with_memory_tracking`（被重写跳过）。这意味着 Sensor 在长时间轮询期间的内存消耗**完全不可观测**。

### 4.4 内存泄漏风险的具体场景

1. **Sensor 函数内部泄漏**：如果 `block_function` 每次调用都创建对象但不释放（如数据库连接、文件句柄），经过多次轮询后内存会持续增长，但 `self.resource_usage` 始终为 `None`，监控无法感知

2. **轮询期间的全局变量膨胀**：`global_vars` 在每次轮询中都被传入，如果 Sensor 函数向 `global_vars` 中添加数据（虽然不建议但技术上可能），这些数据会在后续轮询中累积

3. **长时间运行的线程阻塞**：异步路径下 `parallel=True` 时，`execute_sync` 通过 `run_in_executor` 在线程池中执行。Sensor 的 `time.sleep(60)` 会占用线程池中的线程，如果多个 Sensor 同时运行，可能耗尽线程池

### 4.5 内存跟踪边界总结图

```
┌─ execute_sync ──────────────────────────────────────┐
│  (MemoryManager 已注释，不提供外层跟踪)                │
│                                                       │
│  ┌─ execute_block ─────────────────────────────────┐ │
│  │  ┌─ _execute_block ──────────────────────────┐  │ │
│  │  │  ┌─ SensorBlock.execute_block_function ──┐│  │ │
│  │  │  │                                        ││  │ │
│  │  │  │  from_notebook=True:                   ││  │ │
│  │  │  │   → super() → ✅ 内存跟踪              ││  │ │
│  │  │  │   → ✅ resource_usage 被设置            ││  │ │
│  │  │  │   → ✅ generator 变量流式处理           ││  │ │
│  │  │  │                                        ││  │ │
│  │  │  │  from_notebook=False:                  ││  │ │
│  │  │  │   → while True 循环                    ││  │ │
│  │  │  │   → ❌ 无内存跟踪                      ││  │ │
│  │  │  │   → ❌ resource_usage = None           ││  │ │
│  │  │  │   → ❌ 无 generator 处理               ││  │ │
│  │  │  │   → ❌ 无 delete_variables             ││  │ │
│  │  │  └────────────────────────────────────────┘│  │ │
│  │  └────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  store_variables({})  → 空操作                        │
│  aggregate_summary_info() → 可能扫描残留变量目录       │
│  analyze_outputs({})  → 空操作                        │
│  self._outputs = None  → ✅ 缓存清理                  │
│  self.status = EXECUTED  → 状态更新                    │
│  self.__update_pipeline_block()  → 管道元数据更新      │
└───────────────────────────────────────────────────────┘
```

---

## 五、事实影响汇总

| 维度 | 事实 | 影响 |
|------|------|------|
| **同步路径不跳过 Sensor** | `run_blocks_sync` 没有 `run_sensors` 跳过逻辑 | `run_sensors=False` 时同步路径仍会执行 Sensor，只是降级为单次检查 |
| **`from_notebook` 语义误导** | 实际含义是"是否轮询"，由 `not run_sensors` 推导 | 代码可读性差，容易误解为"来自 Notebook" |
| **Sensor 空输出不触发变量清理** | `variable_mapping = {}` → `store_variables` 空操作 | 历史残留变量不会被清理，`aggregate_summary_info` 可能返回过时数据 |
| **轮询路径无内存跟踪** | `SensorBlock.execute_block_function` 重写跳过 `execute_with_memory_tracking` | `self.resource_usage = None`，长时间轮询的内存消耗不可观测 |
| **轮询路径无 delete_variables** | 基类 generator 逻辑被跳过 | 即使配置了 APPEND/FAIL 策略，也不会在执行前清理变量 |
| **execute_sync 的 MemoryManager 被注释** | 不再有外层内存管理 | 所有块类型都失去了这层保护，Sensor 尤其受影响 |
| **线程池阻塞风险** | `time.sleep(60)` 在 `run_in_executor` 线程中 | 多个 Sensor 并行时可能耗尽线程池 |
| **Pipeline.execute_sync 白名单** | 仅 5 种块类型可作为根块 | Sensor 可作为根块，但 Callback/Conditional 不能 |

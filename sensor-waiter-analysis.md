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

### 2.1 Sensor 空输出的完整代码路径

SensorBlock.execute_block_function 返回 `[]`（空列表），这会沿以下路径影响变量清理：

```
SensorBlock.execute_block_function() → return []  [L4305]
    ↓
_execute_block() → outputs = [] → return []  [L2090]
    ↓
execute_block() → return dict(output=[])  [L1931]
    ↓
execute_sync.__execute() →
    output = dict(output=[])
    block_output = self.post_process_output(output)
    output_count = len(block_output)  # 0
    variable_keys = [f'output_{idx}' for idx in range(0)]  # []
    variable_mapping = dict(zip([], []))  # {}  [L1614]
```

[post_process_output](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1693-L1694) 的实现：
```python
def post_process_output(self, output: Dict) -> List:
    return output['output'] or []  # [] or [] → []
```

**结论**：`output_count = 0`，`variable_mapping = {}`（空字典）。

### 2.2 store_variables 调用链与参数传递

[execute_sync](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1630-L1633) 中调用的是内部函数 `__store_variables`，而不是直接调用 `self.store_variables`：

```python
def __store_variables(
    variable_mapping: Dict[str, Any],
    skip_delete: Optional[bool] = None,
    save_variable_types_only: Optional[bool] = None,
    clean_variable_uuid: Optional[bool] = None,
    ...
) -> Optional[List[Variable]]:
    return self.store_variables(
        variable_mapping,
        clean_variable_uuid=clean_variable_uuid,
        execution_partition=execution_partition,
        override_outputs=override_outputs,  # 来自 execute_sync 参数，默认 True
        skip_delete=skip_delete,
        ...
    )

self._store_variables_in_block_function = __store_variables  # [L1550]
```

然后在 [L1630-1633](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1630-L1633)：
```python
if self._store_variables_in_block_function and isinstance(variable_mapping, dict):
    self._store_variables_in_block_function(variable_mapping)
```

**关键参数传递**：
- `variable_mapping = {}`（空字典）
- `override_outputs = True`（[execute_sync.L1463](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1463) 默认为 True）
- `skip_delete = None`（默认值）
- `override = False`（在 `store_variables` 内部**硬编码**，见下）

### 2.3 store_variables 内部对空 mapping 的处理

[store_variables](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3861) 接收到 `variable_mapping = {}` 时：

#### 步骤 1：调用 __store_variables_prepare

```python
variables_data = self.__store_variables_prepare(
    variable_mapping,         # {}
    execution_partition,
    override=False,           # ← 硬编码为 False！[L3818]
    override_outputs=override_outputs,  # True（来自参数）
    dynamic_block_index=dynamic_block_index,
)
```

#### 步骤 2：__store_variables_prepare 计算 removed_variables

[__store_variables_prepare](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3692-L3722) 的核心逻辑：

```python
def __store_variables_prepare(self, variable_mapping={}, ..., override=False, override_outputs=True):
    
    # 1. 从磁盘读取该 block 下所有现存变量名
    variable_uuids = self.__get_variable_uuids()  
    # 例如：['output_0', 'output_1', 'print_0', 'statistics', 'df']
    
    # 2. 合并变量（对空 mapping 返回 {}）
    variable_mapping = self.__consolidate_variables({})  # {}
    
    # 3. 提取变量名
    variable_names = [clean_name_orig(v) for v in variable_mapping]  # []
    
    # 4. 计算需要删除的变量
    removed_variables = []
    for v in variable_uuids:           # 遍历所有旧变量
        if v in variable_names:        # v in [] → False，不跳过
            continue
        
        is_output_var = is_output_variable(v)  # [utils.py.L289-L304]
        # is_output_variable('output_0') → True
        # is_output_variable('df') → True (include_df 默认 True)
        # is_output_variable('print_0') → False
        # is_output_variable('statistics') → False
        
        # 关键判断（override=False, override_outputs=True）：
        if (override and not is_output_var) or (override_outputs and is_output_var):
            # 简化为：False or is_output_var → is_output_var
            removed_variables.append(v)
    
    return dict(
        removed_variables=['output_0', 'output_1', 'df'],  # 仅输出变量
        variable_mapping={}
    )
```

[is_output_variable](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/utils.py#L289-L304) 的实现：
```python
def is_output_variable(variable_uuid: str, include_df: bool = True) -> bool:
    return (include_df and variable_uuid == 'df') or variable_uuid.startswith('output')
```

**精确结论**：
- **会被删除的变量**：`output_*`（以 output 开头）和 `df`（当 include_df=True 时）
- **不会被删除的变量**：`print_*`、`statistics`、`insights_*`、`suggestions_*`、`metadata` 等非输出变量

#### 步骤 3：判断是否调用 delete_variables

回到 [store_variables.L3853-3859](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3853-L3859)：

```python
if not skip_delete and not is_dynamic_child and variables_data.get('removed_variables'):
    self.delete_variables(
        dynamic_block_index=dynamic_block_index,
        dynamic_block_uuid=dynamic_block_uuid,
        execution_partition=execution_partition,
        variable_uuids=variables_data['removed_variables'],  # 指定删除列表
    )
```

对 Sensor：
- `not skip_delete` → `not None` → `True`
- `not is_dynamic_child` → `not False` → `True`
- `variables_data.get('removed_variables')` → `['output_0', 'output_1', 'df']`（非空）
- **结论**：条件满足，**会调用 delete_variables**

### 2.4 delete_variables 的写入策略分支

[delete_variables](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3724-L3758) 接收到 `variable_uuids=['output_0', 'output_1', 'df']` 时：

```python
def delete_variables(self, variable_uuids=['output_0', 'output_1', 'df'], ...):
    if self.pipeline is None:
        return
    
    variable_uuids_all, block_uuid, _changed = self.__get_variable_uuids(...)
    
    for variable_uuid in variable_uuids or variable_uuids_all:
        variable_object = self.variable_manager.get_variable_object(
            self.pipeline_uuid, block_uuid, variable_uuid, ...
        )
        
        write_policy = (
            self.write_settings.batch_settings.mode
            if self.write_settings and self.write_settings.batch_settings
            else None
        )
        
        # 关键分支：
        if write_policy and variable_object.data_exists():
            if ExportWritePolicy.FAIL == write_policy:
                raise Exception(f'Write policy for block {self.uuid} is {write_policy}.')
            elif ExportWritePolicy.APPEND == write_policy:
                return  # ← 直接 return，后续变量都不删！
        
        variable_object.delete()
```

**各 write_policy 的精确行为**：

| write_policy | 条件满足（有写策略 & 数据存在） | 行为 |
|--------------|-------------------------------|------|
| None（默认） | 不进入 if 分支 | 遍历所有变量，逐个 `variable_object.delete()` |
| FAIL | 进入 if 分支 | 对第一个存在数据的变量抛出异常，终止整个删除流程 |
| APPEND | 进入 if 分支 | 对第一个存在数据的变量执行 `return`，**整个函数终止**，后续变量都不会被删除 |

**Sensor 场景的具体影响**：
- 默认（无 write_policy）：旧的输出变量 `output_0`、`output_1`、`df` 会被正常删除
- FAIL 策略：如果 `output_0` 有数据，直接抛异常，`output_1` 和 `df` 都不会被处理
- APPEND 策略：如果 `output_0` 有数据，直接 `return`，`output_1` 和 `df` 都不会被删除

### 2.5 非输出变量残留的精确说明

**什么是非输出变量**：
- `print_*`：打印变量（通过 `print()` 或 `logger` 生成）
- `statistics`：数据统计信息（行数、列数等）
- `insights_*`：数据洞察（数据类型、缺失值等）
- `suggestions_*`：数据清洗建议
- `metadata`：元数据

**这些变量如何产生**：
- 在 Notebook 模式下运行 Sensor 时，如果 `analyze_outputs=True`，`analyze_outputs` 会为上游数据生成 `statistics`、`insights_*`、`suggestions_*` 等变量
- 块执行过程中的 `print()` 语句会生成 `print_*` 变量
- Sensor 自己的轮询输出 `print('Sensor sleeping for 1 minute...')` 会生成 `print_*` 变量

**这些变量为什么不被清理**：
- 在 `__store_variables_prepare` 中，判断条件是 `(override and not is_output_var) or (override_outputs and is_output_var)`
- `override=False`（硬编码），所以 `(override and not is_output_var)` 恒为 False
- `is_output_var=False`（非输出变量），所以 `(override_outputs and is_output_var)` 也为 False
- 整个条件为 False → 不会被加入 `removed_variables` → 不会被删除

### 2.6 变量清理场景总结

| 场景 | 旧输出变量 (output_*, df) | 旧非输出变量 (print_*, statistics 等) |
|------|--------------------------|-------------------------------------|
| 默认（无 write_policy） | ✅ 被删除 | ❌ 残留 |
| FAIL 策略（有旧数据） | ❌ 抛异常，终止 | ❌ 残留 |
| APPEND 策略（有旧数据） | ❌ 第一个变量触发 return，后续都不删 | ❌ 残留 |
| Notebook 模式（from_notebook=True） | ✅ 被删除（走基类逻辑） | ❌ 残留（同样不满足删除条件） |
| 动态子块（is_dynamic_child=True） | ❌ 不调用 delete_variables（[L3823](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3823)） | ❌ 残留 |

**代码事实**：
1. **空输出会清旧输出变量**——这是之前分析的错误。当 `override_outputs=True`（默认）时，旧的输出变量会被清理
2. **非输出变量始终残留**——因为 `override=False` 硬编码，非输出变量永远不会被 `__store_variables_prepare` 选中删除
3. **APPEND 策略有 bug 级行为**——遇到第一个有数据的变量就 `return`，导致列表后续变量都不被删除

---

## 三、输出分析

### 3.1 analyze_outputs 对空映射的处理

[analyze_outputs](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3397-L3431) 接收到 `variable_mapping = {}` 时：

```python
def analyze_outputs(self, variable_mapping, execution_partition=None, shape_only=False):
    if self.pipeline is None:
        return
    for uuid, data in variable_mapping.items():  # 空字典 → 不进入循环
        if isinstance(data, pd.DataFrame):
            if data.shape[1] > DATAFRAME_ANALYSIS_MAX_COLUMNS or shape_only:
                self.variable_manager.add_variable(
                    ...,
                    variable_type=VariableType.DATAFRAME_ANALYSIS,
                    ...
                )
                continue
            # ... 完整数据分析逻辑
```

**结果**：空映射导致 for 循环不执行，`analyze_outputs` 不产生任何副作用。但调用本身仍然发生。

### 3.2 两种 analyze_outputs 调用模式的差异

[execute_sync.L1650-L1661](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1650-L1661) 中：

```python
if BlockType.CHART != self.type:
    if analyze_outputs:
        self.analyze_outputs(variable_mapping, execution_partition=execution_partition)
    else:
        self.analyze_outputs(variable_mapping, execution_partition=execution_partition, shape_only=True)
```

| 调用场景 | `analyze_outputs` 参数 | `shape_only` | 对 Sensor 的实际效果 |
|---------|----------------------|-------------|---------------------|
| 管道调度（默认） | False | True | 空操作（无数据可分析） |
| Notebook 调试 | True | False | 空操作（无数据可分析） |

**关键事实**：对 Sensor 来说，`analyze_outputs` 的两种调用模式没有任何区别，因为 `variable_mapping = {}`。

### 3.3 aggregate_summary_info 的边界条件

[aggregate_summary_info](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3760-L3774) 在 [execute_sync.L1642-L1645](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1642-L1645) 中被调用：

```python
if not is_dynamic_block_child(self):
    self.aggregate_summary_info()
```

内部实现：
```python
def aggregate_summary_info(self, execution_partition=None):
    if not VARIABLE_DATA_OUTPUT_META_CACHE or not self.variable_manager:
        return
    aggregate_summary_info_for_all_variables(
        self.variable_manager,
        self.pipeline_uuid,
        self.uuid,
        partition=execution_partition,
    )
```

**两个退出条件**：
1. `VARIABLE_DATA_OUTPUT_META_CACHE` 为 False（配置开关）
2. `self.variable_manager` 为 None（无管道）

**Sensor 影响**：即使 Sensor 本次没有输出变量，`aggregate_summary_info_for_all_variables` 仍然会扫描磁盘上的变量目录。如果之前有残留的 `output_0`、`print_0`、`statistics` 等变量文件，它们会被统计到汇总信息中，造成**过时的汇总数据**。

### 3.4 _outputs 缓存重置

```python
self._outputs = None  # [L1648]
```

无论输出是否为空，缓存都会被清除。这是正确的清理行为——确保下次访问 `self.outputs` 时会重新从磁盘加载。

---

## 四、内存记录边界

### 4.1 基类 execute_block_function 的内存跟踪

[Block.execute_block_function.L2152-L2167](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2152-L2167) 在 `MEMORY_MANAGER_V2=True` 时：

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

内存跟踪覆盖：
- 函数执行前的内存快照
- 函数执行后的内存快照
- 计算增量并记录到 `self.resource_usage`

### 4.2 SensorBlock 重写后的内存跟踪丢失

[SensorBlock.execute_block_function.L4274-L4305](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L4274-L4305) 完全重写了基类方法：

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
                print('Sensor sleeping for 1 minute...')
                time.sleep(60)
            return []
```

**两条路径的精确差异**：

| 维度 | `from_notebook=True`（走 super） | `from_notebook=False`（轮询） |
|------|--------------------------------|-------------------------------|
| 内存跟踪 | ✅ `execute_with_memory_tracking` | ❌ 完全缺失（无调用） |
| `self.resource_usage` | 被设置为 `ResourceUsage` 对象 | 停留在初始化值 `None`（[Block.__init__.L444](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L444)） |
| Generator 变量流式存储 | ✅ 逐条存储 + 类型汇总 [L2173-L2242](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2173-L2242) | ❌ 不适用（返回 `[]`） |
| `delete_variables` 调用 | ✅ generator 模式下调用 [L2187-L2191](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2187-L2191) | ❌ 不调用（但 store_variables 会调用，见 2.3 节） |

### 4.3 execute_sync 层面的 MemoryManager

[execute_sync.L1675-L1691](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1675-L1691) 中有一段被注释掉的 `MemoryManager` 上下文：

```python
# if MEMORY_MANAGER_V2:
#     metadata = {}
#     if execution_partition:
#         metadata['execution_partition'] = execution_partition
#     if from_notebook:
#         metadata['origin'] = 'ide'
#     with MemoryManager(
#         scope_uuid=os.path.join(...),
#         process_uuid='block.execute_sync',
#         ...
#     ):
#         return __execute()
return __execute()
```

**现状**：无论 `MEMORY_MANAGER_V2` 是否开启，`execute_sync` 层面都不再有外层内存管理。所有内存跟踪都依赖 `execute_block_function` 内部的 `execute_with_memory_tracking`。

**Sensor 影响**：Sensor 轮询路径既没有 `execute_sync` 层面的 `MemoryManager`（已注释），也没有 `execute_block_function` 层面的 `execute_with_memory_tracking`（被重写跳过）。这意味着 Sensor 在长时间轮询期间的内存消耗**完全不可观测**。

### 4.4 内存跟踪边界总结图

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
│  │  │  │                                        ││  │ │
│  │  │  │  from_notebook=False:                  ││  │ │
│  │  │  │   → while True 循环                    ││  │ │
│  │  │  │   → ❌ 无 execute_with_memory_tracking ││  │ │
│  │  │  │   → ❌ resource_usage = None           ││  │ │
│  │  │  └────────────────────────────────────────┘│  │ │
│  │  └────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                       │
│  store_variables({})                                  │
│   → __store_variables_prepare → removed_variables=[output_*]│
│   → delete_variables(variable_uuids=[output_*])      │
│   → ✅ 旧输出变量被删除                               │
│   → ❌ 非输出变量残留                                 │
│                                                       │
│  aggregate_summary_info()                             │
│   → 扫描磁盘变量目录                                 │
│   → 可能统计到残留变量的过时数据                      │
│                                                       │
│  analyze_outputs({})                                  │
│   → for 循环不执行 → 空操作                           │
│                                                       │
│  self._outputs = None  → ✅ 缓存清理                  │
│  self.status = EXECUTED  → 状态更新                    │
│  self.__update_pipeline_block()  → 管道元数据更新      │
└───────────────────────────────────────────────────────┘
```

---

## 五、事实影响汇总（修正版）

| 维度 | 事实（代码级精确） | 影响 |
|------|-------------------|------|
| **同步路径不跳过 Sensor** | `run_blocks_sync` 没有 `run_sensors` 跳过逻辑 | `run_sensors=False` 时同步路径仍会执行 Sensor，只是降级为单次检查 |
| **`from_notebook` 语义误导** | 实际含义是"是否轮询"，由 `not run_sensors` 推导 | 代码可读性差，容易误解为"来自 Notebook" |
| **空输出会清理旧输出变量** | `override_outputs=True`（默认）+ `__store_variables_prepare` 判断逻辑 | 旧的 `output_*` 和 `df` 变量会被删除（默认无写策略时） |
| **非输出变量始终残留** | `override=False` 硬编码 + `is_output_variable` 过滤逻辑 | `print_*`、`statistics`、`insights_*` 等变量永远不会被清理 |
| **APPEND 策略的异常行为** | `delete_variables` 遇到第一个有数据的变量就 `return` | 后续变量（即使在删除列表中）都不会被删除 |
| **轮询路径无内存跟踪** | `SensorBlock.execute_block_function` 重写跳过 `execute_with_memory_tracking` | `self.resource_usage = None`，长时间轮询的内存消耗不可观测 |
| **execute_sync 的 MemoryManager 被注释** | 不再有外层内存管理 | 所有块类型都失去了这层保护，Sensor 尤其受影响 |
| **线程池阻塞风险** | `time.sleep(60)` 在 `run_in_executor` 线程中 | 多个 Sensor 并行时可能耗尽线程池 |
| **Pipeline.execute_sync 白名单** | 仅 5 种块类型可作为根块 | Sensor 可作为根块，但 Callback/Conditional 不能 |

---

## 六、关键代码引用索引

| 功能 | 文件位置 |
|------|---------|
| store_variables 主入口 | [block/__init__.py#L3776-L3861](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3861) |
| __store_variables_prepare | [block/__init__.py#L3692-L3722](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3692-L3722) |
| override=False 硬编码 | [block/__init__.py#L3818](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3818) |
| delete_variables 写策略分支 | [block/__init__.py#L3724-L3758](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3724-L3758) |
| APPEND 策略 return 语句 | [block/__init__.py#L3755-L3756](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3755-L3756) |
| is_output_variable 定义 | [block/utils.py#L289-L304](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/utils.py#L289-L304) |
| override_outputs 默认 True | [block/__init__.py#L1463](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1463) |
| __store_variables 内部函数 | [block/__init__.py#L1524-L1548](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1524-L1548) |
| delete_variables 调用条件 | [block/__init__.py#L3853-L3859](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3853-L3859) |
| SensorBlock 重写方法 | [block/__init__.py#L4274-L4305](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L4274-L4305) |
| 基类内存跟踪 | [block/__init__.py#L2152-L2167](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2152-L2167) |
| execute_sync MemoryManager 注释 | [block/__init__.py#L1675-L1691](file:///d:/fz/0601/solo-dogfeeding/code/331-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1675-L1691) |

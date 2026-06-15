# Mage AI DAG 解析与拓扑排序实现分析

## 1. 概述

Mage AI 将管道（Pipeline）建模为有向无环图（DAG），节点为 Block（数据加载器、转换器、导出器等），边为 Block 之间的上下游依赖关系。整个系统围绕 **结构读取 → 依赖图构建 → 拓扑排序执行 → 循环检测与错误处理** 四个阶段运转。本文从源码出发，沿调用链逐层核对每个阶段的实现细节。

---

## 2. 核心数据结构

### 2.1 Pipeline 类

[mage_ai/data_preparation/models/pipeline.py](mage_ai/data_preparation/models/pipeline.py) 中的 `Pipeline` 类是 DAG 的顶层容器：

| 属性 | 类型 | 含义 |
|---|---|---|
| `blocks_by_uuid` | `Dict[str, Block]` | 核心执行块的 UUID → Block 映射 |
| `callbacks_by_uuid` | `Dict[str, Block]` | 回调块映射 |
| `conditionals_by_uuid` | `Dict[str, Block]` | 条件块映射 |
| `widgets_by_uuid` | `Dict[str, Block]` | 可视化小部件映射 |
| `extensions` | `Dict[str, Dict]` | 扩展块映射（按 extension_uuid 分组） |
| `block_configs` | `List[Dict]` | 原始 YAML 中的块配置列表 |

### 2.2 Block 类

[mage_ai/data_preparation/models/block/\_\_init\_\_.py](mage_ai/data_preparation/models/block/__init__.py) 中的 `Block` 类通过以下属性构成图的邻接表：

| 属性 | 类型 | 含义 |
|---|---|---|
| `upstream_blocks` | `List[Block]` | 上游依赖块列表（入边） |
| `downstream_blocks` | `List[Block]` | 下游依赖块列表（出边） |
| `conditional_blocks` | `List[Block]` | 关联的条件判断块 |
| `callback_blocks` | `List[Block]` | 关联的回调块 |
| `type` | `BlockType` | 块类型（DATA_LOADER / TRANSFORMER / DATA_EXPORTER / SENSOR / DBT / SCRATCHPAD 等） |

---

## 3. 结构读取方式

### 3.1 YAML 配置文件读取

Pipeline 的配置存储在 `metadata.yaml` 文件中，路径为：

```
{repo_path}/pipelines/{pipeline_uuid}/metadata.yaml
```

核心读取链路：

1. `Pipeline.__init__()`（[pipeline.py:93](mage_ai/data_preparation/models/pipeline.py#L93)）在 `config is None` 时调用 `load_config_from_yaml()`
2. `load_config_from_yaml()`（[pipeline.py:858](mage_ai/data_preparation/models/pipeline.py#L858)）先检查 `catalog_config_path` 是否存在，再调用 `get_config_from_yaml()` 读取 YAML，最终转入 `load_config()`
3. `get_config_from_yaml()`（[pipeline.py:836](mage_ai/data_preparation/models/pipeline.py#L836)）调用 `read_yaml_file(self.config_path)` 完成文件 I/O

```python
def load_config_from_yaml(self):
    catalog = None
    if os.path.exists(self.catalog_config_path):
        catalog = self.get_catalog_from_json()
    self.load_config(self.get_config_from_yaml(), catalog=catalog)
```

### 3.2 异步读取

异步场景通过 `Pipeline.get_async()`（[pipeline.py:584](mage_ai/data_preparation/models/pipeline.py#L584)）实现，使用 `aiofiles` 异步读取 YAML，同时根据 `config.get('type')` 区分普通管道与 Integration 管道，后者额外读取 `catalog.json`。

### 3.3 配置结构

YAML 中每个块的配置包含以下依赖关系字段：

```yaml
blocks:
  - uuid: "my_data_loader"
    type: "data_loader"
    upstream_blocks: []
    downstream_blocks:
      - "my_transformer"
  - uuid: "my_transformer"
    type: "transformer"
    upstream_blocks:
      - "my_data_loader"
    downstream_blocks:
      - "my_exporter"
```

### 3.4 配置校验（加载时唯一显式报错）

`load_config()`（[pipeline.py:864](mage_ai/data_preparation/models/pipeline.py#L864)）在块类型无效时直接抛异常：

```python
if block_type not in [b.value for b in BlockType]:
    raise Exception(
        f'Error loading pipeline ({self.uuid}): Invalid block type ({block_type})',
    )
```

若整个 config 为空，也会在入口处抛出：`raise Exception(f'Invalid pipeline config: {config}')`。

---

## 4. 依赖图构建过程

### 4.1 Block 实例化与邻接表填充

`load_config()` 完成从 YAML 字典到运行时对象的构建，核心步骤如下：

**步骤 1 — 实例化所有 Block 对象**（[pipeline.py:933](mage_ai/data_preparation/models/pipeline.py#L933)）

```python
blocks = [build_shared_args_kwargs(c) for c in self.block_configs]
callbacks = [build_shared_args_kwargs(c) for c in self.callback_configs]
conditionals = [build_shared_args_kwargs(c) for c in self.conditional_configs]
widgets = [build_shared_args_kwargs(c) for c in self.widget_configs]
all_blocks = blocks + callbacks + conditionals + widgets
```

`build_shared_args_kwargs` 函数使用 `BlockFactory.block_class_from_type()` 根据块类型创建对应的 Block 子类实例。

**步骤 2 — 建立上下游引用**

`__initialize_blocks_by_uuid()`（[pipeline.py:1006](mage_ai/data_preparation/models/pipeline.py#L1006)）完成邻接表构建：

```python
all_blocks_by_uuid = {b.uuid: b for b in all_blocks}

for b in configs:
    block = blocks_by_uuid[b['uuid']]
    block.downstream_blocks = [
        all_blocks_by_uuid[uuid]
        for uuid in b.get('downstream_blocks', [])
        if uuid in all_blocks_by_uuid      # ← 缺失节点在此被静默忽略
    ]
    block.upstream_blocks = [
        all_blocks_by_uuid[uuid]
        for uuid in b.get('upstream_blocks', [])
        if uuid in all_blocks_by_uuid      # ← 缺失节点在此被静默忽略
    ]
```

关键设计要点：
- `all_blocks_by_uuid` 将所有类型（block / callback / conditional / widget）合并，确保跨类型引用能正确解析
- `if uuid in all_blocks_by_uuid` 过滤掉不存在的引用——这是缺失节点被静默吞掉的位置（详见第 7 节）

**步骤 3 — 关联回调与条件块**（[pipeline.py:986](mage_ai/data_preparation/models/pipeline.py#L986)）

```python
for block in self.blocks_by_uuid.values():
    block.callback_blocks = blocks_with_callbacks.get(block.uuid, [])
    block.conditional_blocks = blocks_with_conditionals.get(block.uuid, [])
```

**步骤 4 — 执行循环检测验证**（[pipeline.py:1004](mage_ai/data_preparation/models/pipeline.py#L1004)）

```python
self.validate('A cycle was detected in the loaded pipeline')
```

**步骤 5 — 执行框架接管（可选）**

若 `execution_framework` 不为空且匹配已知框架 UUID（[pipeline.py:1015](mage_ai/data_preparation/models/pipeline.py#L1015)），则 `__initialize_blocks_by_uuid()` 会提前返回，由框架自身管理依赖关系：

```python
if execution_framework is not None and ExecutionFrameworkUUID.has_value(execution_framework):
    framework = EXECUTION_FRAMEWORKS_BY_UUID.get(execution_framework)
    framework.initialize_block_instances(blocks_by_uuid)
    return blocks_by_uuid     # 跳过常规邻接表填充
```

### 4.2 动态添加/更新块时的图维护

- `add_block()`（[pipeline.py:1806](mage_ai/data_preparation/models/pipeline.py#L1806)）：先检查 UUID 重复（`raise InvalidPipelineError`），再调用 `__add_block_to_mapping()` 修改内存图，最后调用 `self.validate('A cycle was formed while adding a block')` 验证
- `update_block()`（[pipeline.py:2008](mage_ai/data_preparation/models/pipeline.py#L2008)）：修改块属性后调用 `self.validate('A cycle was formed while updating a block')`

`__add_block_to_mapping()`（[pipeline.py:1784](mage_ai/data_preparation/models/pipeline.py#L1784)）负责：
1. 将新块追加到每个上游块的 `downstream_blocks` 列表
2. 调用 `block.update_upstream_blocks()` 设置新块的上游
3. 支持 `priority` 参数控制块在字典中的插入位置

---

## 5. 执行顺序确定方法（拓扑排序）

Mage AI 采用 **BFS 式拓扑遍历** 而非传统的显式拓扑排序算法，通过运行时依赖检查动态决定执行顺序。存在三条执行路径：

### 5.1 路径一：直接执行（Notebook / CLI）

适用于本地开发场景，由 `Pipeline.execute()` / `Pipeline.execute_sync()` 触发。

**异步执行流程**（[pipeline.py:755](mage_ai/data_preparation/models/pipeline.py#L755)）：

```
Pipeline.execute()
  → 识别 root_blocks（upstream_blocks 为空的块）
  → asyncio.create_task(run_blocks(root_blocks))
```

`run_blocks()` 函数（[block/\_\_init\_\_.py:170](mage_ai/data_preparation/models/block/__init__.py#L170)）核心算法：

```
1. 将所有 root_blocks 放入 Queue，在 tasks 字典中注册 None
2. while Queue 不为空:
   a. 取出队首 block
   b. 跳过不可执行类型（SCRATCHPAD 等在 NON_PIPELINE_EXECUTABLE_BLOCK_TYPES 中）
   c. 防循环保护：tries >= 1000 则 raise Exception
   d. 遍历 block.upstream_blocks:
      - 若 tasks.get(upstream_block.uuid) is None → 上游尚未开始，重新入队，skip=True
   e. 若 skip 则 continue
   f. await asyncio.gather(*[tasks[u.uuid] for u in block.upstream_blocks])
   g. create_block_task(block) → 创建 asyncio.Task，存入 tasks[block.uuid]
   h. 遍历 block.downstream_blocks:
      - 若 downstream_block.uuid not in tasks → 入队并注册 None
3. await asyncio.gather(*remaining_tasks)
```

**同步执行流程** `run_blocks_sync()`（[block/\_\_init\_\_.py:267](mage_ai/data_preparation/models/block/__init__.py#L267)）逻辑类似，但使用 `tasks[block.uuid] = True/False` 代替 `asyncio.Task`，顺序执行而非并行。

**与经典拓扑排序的对比**：

| 特性 | 经典 Kahn 算法 | Mage AI BFS 遍历 |
|---|---|---|
| 入度计算 | 预先计算所有节点入度 | 不预计算，运行时检查 `tasks.get(upstream.uuid) is None` |
| 顺序输出 | 一次性输出完整排序 | 动态发现可执行节点 |
| 并行支持 | 需额外改造 | 天然支持（`asyncio.gather`） |
| 环检测 | 通过剩余入度 > 0 检测 | 通过 tries 计数器（>= 1000）检测 |

### 5.2 路径二：PipelineScheduler 调度执行

适用于生产环境，由 [mage_ai/orchestration/pipeline_scheduler_original.py](mage_ai/orchestration/pipeline_scheduler_original.py) 中的 `PipelineScheduler` 驱动。

**调度流程**：

```
PipelineScheduler.start()
  → PipelineRun.create_block_runs()       # 为所有可执行块创建 BlockRun 记录
  → PipelineScheduler.schedule()
    → update_block_run_statuses(initial_block_runs)  # 传播失败状态
    → PipelineRun.executable_block_runs()             # 筛选当前可执行的 BlockRun
    → 对每个可执行 BlockRun:
        → job_manager.add_job(run_block, ...)         # 提交作业
    → on_block_complete() → 再次 schedule()            # 递归调度
```

**`executable_block_runs()` 方法**（[schedules.py:978](mage_ai/orchestration/db/models/schedules.py#L978)）实现了基于数据库状态的拓扑排序判断：

```
1. _build_block_uuids() → 收集 completed_block_uuids / finished_block_uuids
2. 遍历所有 INITIAL 状态的 BlockRun:
   a. 动态块子节点 → check_all_dynamic_upstreams_completed()
   b. 带有 dynamic_upstream_block_uuids metrics → 检查这些 UUID 是否在 completed 集合中
   c. 数据集成块 → 通过 data_integration_block_uuids_mapping 检查
   d. 常规块 → block.all_upstream_blocks_completed(completed_block_uuids, override)
   e. 若所有上游已完成 → 加入 executable_block_runs
3. 返回可执行列表
```

### 5.3 路径三：Streaming 管道

[mage_ai/data_preparation/executors/streaming_pipeline_executor.py](mage_ai/data_preparation/executors/streaming_pipeline_executor.py) 中的 `StreamingPipelineExecutor` 采用不同的拓扑遍历策略：

**验证阶段** `parse_and_validate_blocks()`（[streaming_pipeline_executor.py:35](mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L35)）：
- 强制要求恰好 1 个 DATA_LOADER 作为 source（且无上游），否则 `raise Exception`
- 每个 TRANSFORMER 恰好有 1 个上游，否则 `raise Exception`
- DATA_EXPORTER 必须是叶子节点（无下游），否则 `raise Exception`

**执行阶段**：从 source 块开始，通过 DFS 递归 `handle_batch_events_recursively()` 沿下游链路逐层处理数据。

### 5.4 ExecutorFactory 执行器选择

[mage_ai/data_preparation/executors/executor_factory.py](mage_ai/data_preparation/executors/executor_factory.py) 根据管道/块的类型和配置选择执行器：

```
PipelineType.PYSPARK       → PySparkPipelineExecutor
ExecutorType.ECS           → EcsPipelineExecutor
ExecutorType.K8S           → K8sPipelineExecutor
PipelineType.STREAMING     → StreamingPipelineExecutor
默认                        → PipelineExecutor (本地 Python)
```

---

## 6. 循环依赖检测

### 6.1 validate() 算法详解

`validate()`（[pipeline.py:2497](mage_ai/data_preparation/models/pipeline.py#L2497)）实现了基于 **DFS + 显式栈** 的环检测算法。

**入口**：合并所有类型块到一个 `combined_blocks` 字典，初始化 `status` 为全 `'unvisited'`。

**遍历起点**：仅从 `self.blocks_by_uuid` 中的块开始（[pipeline.py:2546](mage_ai/data_preparation/models/pipeline.py#L2546)），确保 callback / conditional / widget 等辅助块不会作为 DFS 树根，但会在遍历到时被检查。

**DFS 核心逻辑** `__check_cycle()`（[pipeline.py:2526](mage_ai/data_preparation/models/pipeline.py#L2526)）：

```python
def __check_cycle(block: Block):
    virtual_stack = [StackFrame(block)]
    while len(virtual_stack) > 0:
        frame = virtual_stack[-1]
        if status[frame.uuid] == 'validated':    # 已验证完毕，出栈
            virtual_stack.pop()
            continue
        if not frame.accessed:                    # 首次访问此帧
            if status[frame.uuid] == 'processing':
                # 当前帧正在处理中 → 发现回边 → 环！
                cycle = __print_cycle(frame.uuid, virtual_stack)
                raise InvalidPipelineError(f'{error_msg}: {cycle}')
            frame.accessed = True
            status[frame.uuid] = 'processing'
        if len(frame.children) == 0:             # 无下游，标记已验证
            status[frame.uuid] = 'validated'
            virtual_stack.pop()
        else:
            child_block = combined_blocks[frame.children.pop()]  # 弹出一个下游
            virtual_stack.append(StackFrame(child_block))         # 压栈继续深入
```

**辅助类 `StackFrame`**（[pipeline.py:2560](mage_ai/data_preparation/models/pipeline.py#L2560)）：

```python
class StackFrame:
    def __init__(self, block):
        self.uuid = block.uuid
        self.children = block.downstream_block_uuids   # 复制下游 UUID 列表
        self.accessed = False
```

- `children` 使用 `block.downstream_block_uuids`（属性返回 UUID 字符串列表），每次 `pop()` 弹出一个子 UUID
- 通过 `combined_blocks[uuid]` 查找子 Block 对象入栈

**环路径打印** `__print_cycle()`（[pipeline.py:2518](mage_ai/data_preparation/models/pipeline.py#L2518)）：

从栈中定位环的起始 UUID，截取栈中从起始到末尾的 UUID 序列，用 ` --> ` 拼接。例如 `A --> B --> C --> A`。

**重复 UUID 检测**（[pipeline.py:2550](mage_ai/data_preparation/models/pipeline.py#L2550)）：

```python
check_block_uuids = set()
for config in self.all_block_configs:
    uuid = config.get('uuid')
    if uuid in check_block_uuids:
        raise InvalidPipelineError(
            f'Pipeline is invalid: duplicate blocks with uuid {uuid}'
        )
    check_block_uuids.add(uuid)
```

**异常类 `InvalidPipelineError`**（[pipeline.py:2567](mage_ai/data_preparation/models/pipeline.py#L2567)）继承自 `Exception`，是所有循环/重复检测失败的统一异常类型。

### 6.2 检测触发时机（三处调用点）

| 场景 | 调用位置 | 传入的 error_msg | 完整异常信息示例 |
|---|---|---|---|
| 管道加载 | `load_config()` 末尾（[pipeline.py:1004](mage_ai/data_preparation/models/pipeline.py#L1004)） | `'A cycle was detected in the loaded pipeline'` | `InvalidPipelineError: A cycle was detected in the loaded pipeline: A --> B --> C --> A` |
| 添加块 | `add_block()` 末尾（[pipeline.py:1878](mage_ai/data_preparation/models/pipeline.py#L1878)） | `'A cycle was formed while adding a block'` | `InvalidPipelineError: A cycle was formed while adding a block: X --> Y --> X` |
| 更新块 | `update_block()` 末尾（[pipeline.py:2137](mage_ai/data_preparation/models/pipeline.py#L2137)） | `'A cycle was formed while updating a block'` | `InvalidPipelineError: A cycle was formed while updating a block: X --> Y --> X` |

全局默认消息常量（[pipeline.py:90](mage_ai/data_preparation/models/pipeline.py#L90)）：

```python
CYCLE_DETECTION_ERR_MESSAGE = 'A cycle was detected in this pipeline'
```

注意：三处调用均传入了自定义消息，覆盖了默认常量，因此默认常量实际上不会被使用。

另外，`add_block()` 在调用 `validate()` 之前还有独立的 UUID 重复检查（[pipeline.py:1819](mage_ai/data_preparation/models/pipeline.py#L1819)）：

```python
if block.uuid in all_block_uuids:
    raise InvalidPipelineError(
        f'Block with uuid {block.uuid} already exists in pipeline {self.uuid}'
    )
```

### 6.3 运行时环保护（BFS 遍历中的兜底）

在 `run_blocks()` / `run_blocks_sync()` 中，若 `validate()` 因某种原因未拦截到环（例如图在运行时被修改），则通过 tries 计数器防止无限循环：

```python
# run_blocks() [block/__init__.py:240]
tries_by_block_uuid[block.uuid] += 1
if tries >= 1000:
    raise Exception(f'Block {block.uuid} has tried to execute {tries} times; exiting.')

# run_blocks_sync() [block/__init__.py:298]
tries_by_block_uuid[block.uuid] += 1
if tries >= 1000:
    raise Exception(f'Block {block.uuid} has tried to execute {tries} times; exiting.')
```

这里抛出的是通用 `Exception`，不是 `InvalidPipelineError`，且消息不包含环路径信息——只表示某块被反复调度。

---

## 7. 缺失节点问题

### 7.1 加载时：静默忽略

在 `__initialize_blocks_by_uuid()`（[pipeline.py:1027](mage_ai/data_preparation/models/pipeline.py#L1027)）中，构建上下游引用时使用列表推导式的 `if uuid in all_blocks_by_uuid` 过滤：

```python
block.downstream_blocks = [
    all_blocks_by_uuid[uuid]
    for uuid in b.get('downstream_blocks', [])
    if uuid in all_blocks_by_uuid     # 不存在则静默跳过，不报错不警告
]
block.upstream_blocks = [
    all_blocks_by_uuid[uuid]
    for uuid in b.get('upstream_blocks', [])
    if uuid in all_blocks_by_uuid     # 同上
]
```

**效果**：YAML 中引用了不存在的块 UUID 不会导致加载失败，该引用被丢弃。这造成两个方向的影响：
- 块 A 声明了 `upstream_blocks: [X]`，但 X 不存在 → A 的 `upstream_blocks` 列表为空 → A 变成 root block，可能在没有输入的情况下执行
- 块 A 声明了 `downstream_blocks: [Y]`，但 Y 不存在 → A 的 `downstream_blocks` 列表少了一个条目 → Y 不会被自动调度

### 7.2 运行时：get_block() 的 print 错误

`get_block()`（[pipeline.py:1882](mage_ai/data_preparation/models/pipeline.py#L1882)）在找不到块时有两层回退逻辑：

```python
block = mapping.get(block_uuid)
if not block:
    # 回退1: 按 ':' 分割取前缀，适配动态块/数据集成块/复制块的 UUID 格式
    block = mapping.get(block_uuid.split(':')[0])

if not block:
    # 回退2: 所有方式都找不到 → 仅 print 到 stdout，不抛异常
    print(
        f'[ERROR] Pipeline.get_block: '
        f'block {block_uuid} with type {block_type} does not exist in '
        f'pipeline {self.uuid} for repo_path {self.repo_path}.'
    )

return block   # 返回 None
```

关键问题：回退2仅 `print` 到 stdout，**不写入结构化日志、不抛异常、不返回错误**。调用方如果不检查返回值，后续操作会抛 `AttributeError: 'NoneType' object has no attribute ...`。

### 7.3 调度器中的缺失处理

`executable_block_runs()`（[schedules.py:1078](mage_ai/orchestration/db/models/schedules.py#L1078)）在遍历初始块运行时，通过 `pipeline.get_block(block_run.block_uuid)` 获取块：

```python
for block_run in self.initial_block_runs:
    block = pipeline.get_block(block_run.block_uuid)

    if block is not None and block.is_dynamic_streaming:
        ...
        continue

    # ... 后续逻辑中 block 可能为 None
```

当 `block is None` 时，代码跳过了动态流式块的分支，但后续的常规块判断（[schedules.py:1176](mage_ai/orchestration/db/models/schedules.py#L1176)）中有：

```python
completed = (
    not incomplete
    and block is not None       # ← block 为 None 时 completed = False
    and block.all_upstream_blocks_completed(...)
)
```

因此 `block is None` 的 BlockRun 永远不会被标记为 executable，最终作为 INITIAL 状态残留，不会被执行也不会被报错。

### 7.4 缺失节点风险总结

| 场景 | 行为 | 是否有错误提示 |
|---|---|---|
| YAML 中引用不存在的上游/下游 UUID | 引用被静默丢弃 | ❌ 无 |
| `get_block()` 找不到块 | `print` 到 stdout，返回 `None` | ⚠️ 仅 stdout `[ERROR]`，非结构化日志 |
| 调度器中块为 None | BlockRun 永远不执行 | ❌ 无 |
| 块类型无效 | `raise Exception('Invalid block type')` | ✅ 有 |
| 块 UUID 重复 | `raise InvalidPipelineError('duplicate blocks')` | ✅ 有 |

---

## 8. 动态配置

### 8.1 动态块（Dynamic Block）

Mage AI 支持动态块，可在运行时根据上游输出动态生成子块实例：

- `is_dynamic_block(block)` — 判断是否为动态块
- `is_dynamic_block_child(block)` — 判断是否为动态块的子块
- `should_reduce_output(block)` — 是否需要将多个子块输出聚合

动态块的子块在运行时才被创建（通过 `DynamicBlockFactory`），UUID 格式为 `parent_uuid:0`、`parent_uuid:1` 等。

### 8.2 执行器类型动态渲染

Pipeline 的 `executor_type` 支持 Jinja2 模板变量（[pipeline.py:1985](mage_ai/data_preparation/models/pipeline.py#L1985)）：

```python
def get_executor_type(self) -> Optional[str]:
    if self._rendered_executor_type is None:
        if self.executor_type:
            return Template(self.executor_type).render(**get_template_vars())
```

### 8.3 执行框架

当 `execution_framework` 不为空且匹配已知框架 UUID 时（[pipeline.py:1015](mage_ai/data_preparation/models/pipeline.py#L1015)），依赖关系由框架自身管理，`__initialize_blocks_by_uuid()` 会跳过常规邻接表填充。

### 8.4 数据集成管道

Integration 管道具有特殊的拓扑结构：
- Source 块产生多个 stream
- 每个 stream 可创建子控制器块（controller child）
- 支持 `run_in_parallel` 配置并行/串行执行
- 子块 UUID 格式：`[block UUID]:[source UUID]:[stream]:[index]`

---

## 9. 调度失败提示路径

调度失败沿着 **块级 → 管道级 → 通知** 三层传播，以下是完整调用链。

### 9.1 块运行状态机

```
INITIAL → QUEUED → RUNNING → COMPLETED
                           → FAILED
                           → CONDITION_FAILED
                           → UPSTREAM_FAILED
                           → CANCELLED
```

### 9.2 块执行失败 → FAILED

**触发路径**：`BlockExecutor.execute()`（[block_executor.py:648](mage_ai/data_preparation/executors/block_executor.py#L648)）

```
BlockExecutor.execute()
  → __execute_with_retry() 抛出异常
  → except Exception as error:
      ├── on_failure callback (如果提供)
      │     → on_block_failure(block_uuid, error=error_details)
      └── __update_block_run_status(FAILED, error_details=dict(error=error))
            → BlockRun.update(status=FAILED)
            若 DB 更新失败 → 回退到 callback_url HTTP PUT
```

`__update_block_run_status()`（[block_executor.py:1369](mage_ai/data_preparation/executors/block_executor.py#L1369)）的完整逻辑：
1. 优先通过 `BlockRun.query.get(block_run_id)` 获取并更新 DB 记录
2. 若 DB 操作失败，回退到 `requests.put(callback_url, ...)` 发送 HTTP 请求
3. 若两者都失败，`logger.exception('Failed to update block run status ...')`

### 9.3 条件块返回 False → CONDITION_FAILED

**触发路径**：`BlockExecutor._execute_conditional()`（[block_executor.py:1200](mage_ai/data_preparation/executors/block_executor.py#L1200)）

```
BlockExecutor.execute()
  → _execute_conditional()
    → 遍历 block.conditional_blocks:
        → conditional_block.execute_conditional()
        → 若返回 False 或抛异常 → result = False
  → 若 conditional_result 为 False:
      ├── 数据集成块: 递归 __update_condition_failed() 遍历下游
      │     → 对每个下游块: __update_block_run_status(CONDITION_FAILED)
      └── 常规块: __update_block_run_status(CONDITION_FAILED)
      → 日志: 'Conditional block(s) returned false for {uuid}. '
              'This block run and downstream blocks will be set as CONDITION_FAILED.'
      → return dict(output=[])
```

### 9.4 上游失败传播 → UPSTREAM_FAILED

**触发路径**：`PipelineRun.update_block_run_statuses()`（[schedules.py:1223](mage_ai/orchestration/db/models/schedules.py#L1223)）

在每次 `schedule()` 开头被调用：

```python
# pipeline_scheduler_original.py:617
self.pipeline_run.update_block_run_statuses(self.pipeline_run.initial_block_runs)
```

算法逻辑：

```
1. 收集 failed_block_uuids = {所有 FAILED 或 UPSTREAM_FAILED 的 block_uuid}
2. 收集 condition_failed_block_uuids = {所有 CONDITION_FAILED 的 block_uuid}
3. 遍历所有 INITIAL 状态的 BlockRun:
   a. 获取其 upstream_block_uuids（优先从 metrics.dynamic_upstream_block_uuids 取）
   b. 若 upstream 与 failed 集合有交集 → 更新为 UPSTREAM_FAILED
   c. 若 upstream 与 condition_failed 集合有交集 → 更新为 CONDITION_FAILED
   d. 若未更新 → 加入 not_updated_block_runs
4. 若本轮有更新 → self.refresh() → 递归调用 update_block_run_statuses(not_updated_block_runs)
```

递归保证：失败状态会沿 DAG 逐层传播到所有间接下游。

### 9.5 块级失败 → PipelineScheduler.on_block_failure()

**触发路径**：[pipeline_scheduler_original.py:443](mage_ai/orchestration/pipeline_scheduler_original.py#L443)

```
on_block_failure(block_uuid, **kwargs)
  → BlockRun.get() 获取 block_run
  → 将 error 信息写入 metrics['error'] = {error, errors, message}
  → block_run.update(status=FAILED)
  → logger.exception('BlockRun {id} (block_uuid: {uuid}) failed.')
  → 若不允许 allow_blocks_to_fail 且为 Integration 管道:
      → kill_pipeline_run_job()
      → kill_integration_stream_job() 对每个 stream
```

### 9.6 管道级失败 → on_pipeline_run_failure()

**触发路径**：`schedule()` 方法中的三个入口（[pipeline_scheduler_original.py:260](mage_ai/orchestration/pipeline_scheduler_original.py#L260)、[pipeline.py:307](mage_ai/data_preparation/models/pipeline.py#L307)、[pipeline_scheduler_original.py:309](mage_ai/orchestration/pipeline_scheduler_original.py#L309)）：

1. **所有块完成但有失败的**：`any_blocks_failed() and not allow_blocks_to_fail`
2. **管道超时**：`__check_pipeline_run_timeout()` 返回 True
3. **存在失败块**：`any_blocks_failed() and not allow_blocks_to_fail`

```
on_pipeline_run_failure(error_msg)
  → UsageStatisticLogger().pipeline_run_ended_sync()
  → 收集失败块错误信息:
      → 遍历 failed_block_runs
      → 取 br.metrics['error']['message']
      → 截断到最后 50 行（超过 50 行插入 '... (error truncated)'）
      → 拼接 stacktrace = f'Error for block {br.block_uuid}:\n{message}'
  → notification_sender.send_pipeline_run_failure_message(
        pipeline, pipeline_run, error=error_msg, stacktrace=stacktrace
    )
  → cancel_block_runs_and_jobs(pipeline_run, pipeline)
      → BlockRun.batch_update_status(ids, CANCELLED)
      → job_manager.kill_pipeline_run_job()
      → 若为 Integration/Streaming: kill_integration_stream_job() 每个 stream
```

### 9.7 失败通知消息格式

`notification_sender.send_pipeline_run_failure_message()` 发送的通知包含：

| 字段 | 来源 |
|---|---|
| `error` | `'Failed blocks: {uuid1}, {uuid2}.'` 或 `'Pipeline run timed out.'` |
| `stacktrace` | `'Error for block {block_uuid}:\n{truncated_message}'`（最多 50 行） |
| `pipeline` | Pipeline 对象 |
| `pipeline_run` | PipelineRun 对象 |

### 9.8 重试机制

Block 执行支持可配置的重试策略（[block_executor.py:594](mage_ai/data_preparation/executors/block_executor.py#L594)），合并管道级和块级配置：

```python
retry_config = merge_dict(
    self.pipeline.repo_config.retry_config or dict(),
    self.block.retry_config or dict(),
)
```

`RetryConfig` 包含：`retries`、`delay`、`max_delay`、`exponential_backoff`。重试耗尽后异常向上传播，进入 9.2 的 FAILED 流程。

---

## 10. 模块间协作关系

```
┌───────────────────────────────────────────────────────────────┐
│                     Pipeline (DAG Container)                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐ │
│  │ blocks_by  │  │callbacks  │  │conditionals│  │ widgets   │ │
│  │ _uuid      │  │_by_uuid   │  │_by_uuid   │  │_by_uuid   │ │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘ │
│        │              │              │              │         │
│        └──────────────┴──────┬───────┴──────────────┘         │
│                              │                                 │
│                    validate() (环检测)                         │
│                    load_config() (图构建)                      │
│                    get_block() (运行时查找)                     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──────┐ ┌──────▼───────┐ ┌─────▼────────┐
     │ 直接执行路径   │ │ 调度器执行路径│ │ 流式执行路径  │
     │               │ │              │ │              │
     │ execute()     │ │ Pipeline     │ │ Streaming    │
     │ execute_sync()│ │ Scheduler    │ │ Pipeline     │
     │               │ │              │ │ Executor     │
     │ run_blocks()  │ │ PipelineRun  │ │              │
     │ run_blocks_   │ │ .executable_ │ │ DFS 递归     │
     │ sync()        │ │ block_runs() │ │ 遍历         │
     └───────┬───────┘ └──────┬───────┘ └──────┬───────┘
             │                │                │
             └────────────────┼────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  BlockExecutor    │
                    │  + 重试机制       │
                    │  + 条件评估       │
                    │  + 回调触发       │
                    │  + 状态更新       │
                    └───────────────────┘
```

---

## 11. 主流程总结

### 11.1 管道加载与图构建流程

```
1. Pipeline.__init__()
2. load_config_from_yaml()
   ├── get_config_from_yaml() → 读取 metadata.yaml
   └── load_config(config)
       ├── 校验 config 非空
       ├── 校验 block_type 合法（否则 raise Exception）
       ├── BlockFactory 创建 Block 实例
       ├── __initialize_blocks_by_uuid()
       │     ├── 合并 all_blocks_by_uuid
       │     ├── 填充 upstream/downstream 引用（缺失节点静默跳过）
       │     └── 执行框架可选接管
       ├── 关联回调/条件块
       └── validate('A cycle was detected in the loaded pipeline')
             ├── DFS 三色标记法检测环 → InvalidPipelineError
             └── 重复 UUID 检测 → InvalidPipelineError
```

### 11.2 直接执行流程

```
1. Pipeline.execute() / execute_sync()
2. 识别 root_blocks (upstream_blocks 为空)
3. run_blocks(root_blocks) / run_blocks_sync(root_blocks)
4. BFS 遍历:
   ├── tries >= 1000 → raise Exception (环保护)
   ├── 上游未开始 → 重新入队
   ├── await 上游任务 / 检查上游状态
   ├── 执行当前块 + 运行测试
   └── 下游块入队
5. await 所有剩余任务
```

### 11.3 调度器执行流程

```
1. PipelineScheduler.start()
   → PipelineRun.create_block_runs()      # 为所有可执行块创建 BlockRun
2. PipelineScheduler.schedule()
   → update_block_run_statuses()          # 传播 FAILED/CONDITION_FAILED
   → executable_block_runs()              # 基于 DB 状态筛选可执行块
   → job_manager.add_job(run_block, ...)  # 提交作业
3. on_block_complete() → schedule()       # 递归调度
   或 on_block_failure() → 更新状态为 FAILED
4. 判断管道完成/失败
   → on_pipeline_run_failure()            # 通知 + 取消剩余作业
   或 pipeline_run.complete()             # 成功完成
```

---

## 12. 潜在风险

| 风险类别 | 描述 | 来源代码位置 | 严重程度 |
|---|---|---|---|
| **缺失节点静默忽略** | YAML 中引用不存在的块 UUID 不会报错，仅被列表推导式过滤，上游丢失使块意外成为 root block | [pipeline.py:1032](mage_ai/data_preparation/models/pipeline.py#L1032) | 高 |
| **get_block() 仅 print 不抛异常** | 找不到块时 `print` 到 stdout 后返回 `None`，调用方若不检查则后续 AttributeError | [pipeline.py:1913](mage_ai/data_preparation/models/pipeline.py#L1913) | 高 |
| **图结构修改无事务** | `add_block()` / `update_block()` 中先修改内存结构再 validate，环检测失败时内存状态已被修改 | [pipeline.py:1878](mage_ai/data_preparation/models/pipeline.py#L1878)、[pipeline.py:2137](mage_ai/data_preparation/models/pipeline.py#L2137) | 中 |
| **运行时环检测滞后** | `run_blocks()` 中通过 tries >= 1000 检测环，最多需要 1000 次无效迭代才能发现，且异常消息不含环路径 | [block/\_\_init\_\_.py:242](mage_ai/data_preparation/models/block/__init__.py#L242) | 中 |
| **BlockRun 永久 INITIAL** | 调度器中 `block is None` 的 BlockRun 永远不执行也不报错，最终残留为 INITIAL | [schedules.py:1176](mage_ai/orchestration/db/models/schedules.py#L1176) | 中 |
| **动态块 UUID 冲突** | 动态子块使用 `parent_uuid:index` 格式，`get_block()` 用 `split(':')[0]` 回退查找，若用户块 UUID 包含 `:` 可能导致误匹配 | [pipeline.py:1910](mage_ai/data_preparation/models/pipeline.py#L1910) | 低 |
| **Streaming 管道结构限制** | 强制单 source、单上游 transformer 的约束过于严格，无法表达更复杂的 DAG | [streaming_pipeline_executor.py:35](mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L35) | 低 |

---

## 13. 后续研究方向

1. **缺失节点校验增强**：在 `__initialize_blocks_by_uuid()` 中对缺失的引用产生警告日志（而非静默跳过），帮助用户及早发现配置错误

2. **get_block() 异常化**：将 `print` 改为 `logger.error()` 并在关键调用路径中检查返回值，或提供 `get_block_strict()` 变体在找不到块时抛异常

3. **增量环检测**：当前 `validate()` 每次全量遍历，对于大型管道可考虑增量检测算法，仅在修改影响范围内检查

4. **图结构修改事务化**：将 `validate()` → `save()` 的过程包装为原子操作，验证失败时回滚内存状态（深拷贝 → 修改 → validate → 赋值）

5. **BFS 环检测优化**：`run_blocks()` 中的 tries >= 1000 兜底可替换为显式入度计数，在 O(V+E) 时间内判断是否有环，并输出具体环路径

6. **BlockRun 超时清理**：对调度器中长期处于 INITIAL 状态的 BlockRun 增加 gc 逻辑或告警，避免因缺失节点导致管道永远无法完成

7. **并行度自适应**：当前 BFS 遍历中同一层的块全部并行执行，可考虑根据系统资源（CPU/内存）和 `concurrency_config.block_run_limit` 动态控制并行度

8. **DAG 可视化与调试**：利用 [presenters/blocks/graph.py](mage_ai/presenters/blocks/graph.py) 中已有的图构建逻辑，增强运行时 DAG 可视化，展示动态块展开后的完整执行图

9. **拓扑排序缓存**：对于结构不变的管道，可缓存拓扑排序结果，避免每次执行都重新遍历

10. **Streaming 管道的 DAG 扩展**：放宽单 source 限制，支持多源流式 DAG，适应更复杂的数据流场景

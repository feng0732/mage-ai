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

调度失败沿着 **块级状态更新 → 管道级检测 → 通知发送** 三层传播，以下是沿源码核对的完整调用链。

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

在每次 `schedule()` 开头被调用（[pipeline_scheduler_original.py:617](mage_ai/orchestration/pipeline_scheduler_original.py#L617)）：

```python
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

### 9.5 块级失败回调 → PipelineScheduler.on_block_failure()

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

**重要**：`on_block_failure()` **不直接触发** `on_pipeline_run_failure()`。它只更新块状态 + 日志 + 可选 kill 作业。管道级失败由下一次 `schedule()` 调用时通过 `any_blocks_failed()` 检查发现。

### 9.6 管道级失败：5 个触发入口

管道级失败共有 **5 个独立入口**，分为三大类清理路径：
- **A 类（经 on_pipeline_run_failure 统一处理）**：入口 2、3、4 — 先由调用方更新管道状态，再调用统一函数
- **B 类（经 stop → stop_pipeline_run 清理）**：入口 5 — 先 stop 清理，再直接发通知
- **C 类（直调通知 + 无清理）**：入口 1 — 更新状态后直接发通知，不清理

#### 入口 1：start() 初始化失败（C 类，无清理，直调通知）

**位置**：[pipeline_scheduler_original.py:176](mage_ai/orchestration/pipeline_scheduler_original.py#L176)

**触发条件**：`start()` 中 `create_block_runs()`（非集成管道）或 `initialize_state_and_runs()`（集成管道）抛异常。

**完整代码路径**：

```python
except Exception as e:
    error_msg = 'Fail to initialize block runs.'
    self.logger.exception(error_msg, ...)                  # 1. 日志记录
    self.pipeline_run.update(status=PipelineRunStatus.FAILED)  # 2. 管道状态→FAILED
    self.notification_sender.send_pipeline_run_failure_message(  # 3. 直打发通知
        pipeline=self.pipeline,
        pipeline_run=self.pipeline_run,
        error=error_msg,                                     # 仅传 error，无 stacktrace
    )
    return False
finally:
    lock.release_lock(lock_key)                             # 4. finally 仅释放锁
```

**error**：`'Fail to initialize block runs.'`

**清理行为**（全 ❌）：

| 清理动作 | 是否执行 | 原因 |
|---|---|---|
| `stop_pipeline_run()` | ❌ 无 | 代码中未调用 |
| `cancel_block_runs_and_jobs()` | ❌ 无 | 未经过任何清理路径 |
| `UsageStatisticLogger.pipeline_run_ended_sync()` | ❌ 无 | 未记录结束统计 |
| `BlockRun.batch_update_status(CANCELLED)` | ❌ 无 | 未调用 |
| `kill_pipeline_run_job()` | ❌ 无 | 未调用 |

**实际风险分析**：
- 此异常发生在 `start()` 阶段，管道尚未真正调度。如果是 `create_block_runs()` 抛异常，BlockRun 可能尚未创建或部分创建，没有正在运行的作业需要 kill，因此**在普通管道场景下实际影响有限**
- **但**：如果是集成管道的 `initialize_state_and_runs()` 在创建了部分 state 文件或 output 文件后抛异常，这些残留文件不会被清理
- 真正缺失的是 `UsageStatisticLogger` 统计记录——该管道的结束状态不会被上报到使用统计

#### 入口 2：所有块完成 + 存在失败块（A 类，经 on_pipeline_run_failure）

**位置**：[pipeline_scheduler_original.py:250](mage_ai/orchestration/pipeline_scheduler_original.py#L250)

**触发条件**：
```python
if self.pipeline_run.all_blocks_completed(self.allow_blocks_to_fail):
    if self.pipeline_run.any_blocks_failed():
        # → 触发失败
    else:
        # → 成功完成
```

其中：
- `all_blocks_completed(include_failed_blocks)`（[schedules.py:1522](mage_ai/orchestration/db/models/schedules.py#L1522)）：默认认为 COMPLETED + CONDITION_FAILED 是"完成"状态；若 `include_failed_blocks=True`，则 FAILED + UPSTREAM_FAILED 也算完成
- 当 `allow_blocks_to_fail=True` 时，`include_failed_blocks=True`，意味着即使有块失败，也要等所有块都跑完才判定管道结束
- `any_blocks_failed()`（[schedules.py:1519](mage_ai/orchestration/db/models/schedules.py#L1519)）：`any(b.status == FAILED for b in self.block_runs)`，只检查 FAILED 状态（不含 UPSTREAM_FAILED）

**调用前置动作**（调用 on_pipeline_run_failure 之前）：
```python
self.pipeline_run.update(
    status=PipelineRunStatus.FAILED,
    completed_at=datetime.now(tz=pytz.UTC),
)
```

**error_msg**：`'Failed blocks: {uuid1}, {uuid2}.'`（列出所有 FAILED 状态块的 UUID）

#### 入口 3：管道超时（A 类，经 on_pipeline_run_failure）

**位置**：[pipeline_scheduler_original.py:302](mage_ai/orchestration/pipeline_scheduler_original.py#L302)

**触发条件**：`__check_pipeline_run_timeout()` 返回 True。

**调用前置动作**：
```python
status = self.pipeline_schedule.timeout_status or PipelineRunStatus.FAILED
self.pipeline_run.update(status=status)
```

**error_msg**：`'Pipeline run timed out.'`

**status**：取 `self.pipeline_schedule.timeout_status`，默认 `PipelineRunStatus.FAILED`。**只有当 status == FAILED 时才发送通知**，但无论如何都会取消作业。

#### 入口 4：存在失败块 + 不允许失败（A 类，经 on_pipeline_run_failure）

**位置**：[pipeline_scheduler_original.py:308](mage_ai/orchestration/pipeline_scheduler_original.py#L308)

**触发条件**：`any_blocks_failed() and not self.allow_blocks_to_fail`

与入口 2 的区别：入口 2 是**所有块都跑完后**才判定；入口 4 是**只要有块失败且不允许失败**，就立即判定管道失败（不等其他块跑完）。

**调用前置动作**：
```python
self.pipeline_run.update(status=PipelineRunStatus.FAILED)
```

**error_msg**：`'Failed blocks: {uuid1}, {uuid2}.'`

#### 入口 5：内存超限（B 类，经 stop → stop_pipeline_run 清理）

**位置**：[pipeline_scheduler_original.py:490](mage_ai/orchestration/pipeline_scheduler_original.py#L490)

**触发条件**：`memory_usage_failure()` 被调用（内存使用达到 `MEMORY_USAGE_MAXIMUM * 100%` 上限）。

**完整代码路径**：

```python
def memory_usage_failure(self, tags=None):
    msg = 'Memory usage across all pipeline runs has reached or exceeded ...'
    self.logger.info(msg, **tags)           # 1. 日志
    self.stop()                               # 2. 完整清理（关键！）
    self.notification_sender.send_pipeline_run_failure_message(  # 3. 直接发通知
        pipeline=self.pipeline,
        pipeline_run=self.pipeline_run,
        summary=msg,                          # 传 summary，而非 error/stacktrace
    )
    if INTEGRATION:
        calculate_pipeline_run_metrics(...)
```

**stop() 调用链**（[pipeline_scheduler_original.py:206](mage_ai/orchestration/pipeline_scheduler_original.py#L206)）：

```python
def stop(self):
    stop_pipeline_run(self.pipeline_run, self.pipeline)
```

**summary**：`'Memory usage across all pipeline runs has reached or exceeded the maximum limit of {int(MEMORY_USAGE_MAXIMUM * 100)}%.'`

**清理行为**（通过 stop_pipeline_run 全 ✅）：见 9.7 节对比表。

### 9.7 清理路径对比：stop_pipeline_run vs on_pipeline_run_failure vs 无清理

#### stop_pipeline_run() 函数详解

**位置**：[pipeline_scheduler_original.py:1426](mage_ai/orchestration/pipeline_scheduler_original.py#L1426)

```python
def stop_pipeline_run(pipeline_run, pipeline=None, status=CANCELLED):
    if pipeline_run.status not in [INITIAL, RUNNING]:  # 前置状态守卫
        return
    pipeline_run.update(status=status)                 # 更新管道状态
    UsageStatisticLogger().pipeline_run_ended_sync(pipeline_run)  # 上报使用统计
    cancel_block_runs_and_jobs(pipeline_run, pipeline)  # 取消块和作业
```

**关键细节**：
- **前置守卫**：`status not in [INITIAL, RUNNING]` 时直接 return，不做任何清理。因此如果管道状态已被设置为 FAILED，再调用 stop_pipeline_run() 会直接退出
- **默认状态是 CANCELLED**：不是 FAILED。这意味着内存超限导致的失败，管道最终状态是 CANCELLED

#### cancel_block_runs_and_jobs() 函数详解

**位置**：[pipeline_scheduler_original.py:1460](mage_ai/orchestration/pipeline_scheduler_original.py#L1460)

```python
def cancel_block_runs_and_jobs(pipeline_run, pipeline=None):
    # 1. 批量更新块运行状态
    for b in pipeline_run.block_runs:
        if b.status in [INITIAL, QUEUED, RUNNING]:
            → 加入取消列表
        if b.status == RUNNING:
            → 加入运行中列表
    BlockRun.batch_update_status(ids, CANCELLED)

    # 2. kill 作业（两种策略）
    if pipeline and (
        pipeline.type in [INTEGRATION, STREAMING]
        or pipeline.run_pipeline_in_one_process
    ):
        # 策略 A：整体 kill（单作业模式）
        job_manager.kill_pipeline_run_job(pipeline_run.id)
        if INTEGRATION:  # 额外 kill 每个 stream 的作业
            for stream in pipeline.streams():
                job_manager.kill_integration_stream_job(...)
        if K8S executor:
            ExecutorFactory.get_pipeline_executor(...).cancel(...)
    else:
        # 策略 B：逐个 kill（多作业模式）
        for b in running_blocks:
            job_manager.kill_block_run_job(b.id)

    # 3. 异步兜底：再次确保所有已取消的作业都被杀掉
    GenericJob.enqueue_cancel_pipeline_run(pipeline_run.id, cancelled_ids)
```

#### 三类清理路径的完整对比

| 清理动作 | A 类（入口2/3/4）<br>on_pipeline_run_failure | B 类（入口5）<br>stop_pipeline_run | C 类（入口1）<br>直调通知无清理 |
|---|---|---|---|
| **管道状态更新** | ✅ 调用方在**调用前**更新<br>（status 由传入参数决定） | ✅ 在函数内更新为 **CANCELLED**<br>（可通过参数覆盖） | ✅ 在通知前更新为 **FAILED** |
| **前置状态守卫** | ❌ 无，直接执行 | ✅ 有（非 INITIAL/RUNNING 直接 return） | ❌ 无 |
| `UsageStatisticLogger`<br>`.pipeline_run_ended_sync()` | ✅ 在函数开头调用 | ✅ 在 cancel 前调用 | ❌ 完全缺失 |
| `send_pipeline_run_failure_message()` | ✅ 仅当 `status == FAILED` 时<br>传 `error=` + 自动构建 `stacktrace=` | ✅ 在 stop 之后**直调**<br>传 `summary=`（无 error/stacktrace） | ✅ 在 finally 之前**直调**<br>传 `error=`（无 stacktrace） |
| `cancel_block_runs_and_jobs()` | ✅ 在通知之后调用<br>传入 `(pipeline_run, pipeline)` | ✅ 通过 stop 间接调用<br>传入 `(pipeline_run, pipeline)` | ❌ 完全缺失 |
| `BlockRun.batch_update(CANCELLED)` | ✅ 含 | ✅ 含 | ❌ 无 |
| `kill_pipeline_run_job()`<br>（整体/集成/流式） | ✅ 含 | ✅ 含 | ❌ 无 |
| `kill_block_run_job()`<br>（逐个块） | ✅ 含 | ✅ 含 | ❌ 无 |
| `GenericJob.enqueue_cancel_`<br>`pipeline_run()` | ✅ 含（异步兜底） | ✅ 含（异步兜底） | ❌ 无 |

**关键结论**：

1. **入口 5（内存超限）的清理实际上是完整的**——通过 `stop() → stop_pipeline_run() → cancel_block_runs_and_jobs()` 执行了所有清理动作。之前的"不会触发 cancel_block_runs_and_jobs"判断是错误的，真正缺失清理的是入口 1。

2. **最终管道状态不同**：
   - A 类（入口2/3/4）：最终状态是 **FAILED**（由各调用方在调用前手动 update）
   - B 类（入口5）：最终状态是 **CANCELLED**（stop_pipeline_run 默认参数）
   - C 类（入口1）：最终状态是 **FAILED**（在 except 块中手动 update）

3. **通知内容差异**：
   - A 类：自动构建 stacktrace（取第一个有错误消息的失败块）
   - B 类：无 stacktrace，使用 summary 作为通知主体
   - C 类：无 stacktrace，只有固定的 error 文本

4. **stop_pipeline_run 的状态守卫陷阱**：如果先调用 `pipeline_run.update(status=FAILED)` 再调用 `stop()`，stop_pipeline_run() 会因 `status not in [INITIAL, RUNNING]` 直接 return，**不做任何清理**。这是一条潜在的代码路径陷阱，但当前5个入口的调用顺序都避开了这个陷阱。

### 9.8 on_pipeline_run_failure() 统一处理流程

**位置**：[pipeline_scheduler_original.py:341](mage_ai/orchestration/pipeline_scheduler_original.py#L341)

仅入口 2、3、4 会走到这里。入口 1 走 C 类（无清理），入口 5 走 B 类（stop 清理）。

```
on_pipeline_run_failure(error_msg, status=FAILED)
  → UsageStatisticLogger().pipeline_run_ended_sync()
  → 若 status == FAILED:
      → 收集失败块错误信息（详见 9.9）
      → notification_sender.send_pipeline_run_failure_message(
            pipeline, pipeline_run, error=error_msg, stacktrace=stacktrace
        )
  → cancel_block_runs_and_jobs(pipeline_run, pipeline)
      → BlockRun.batch_update_status(ids, CANCELLED)
      → kill_pipeline_run_job()（集成/流式/单进程模式）
      → kill_block_run_job()（普通模式，逐个运行块）
      → 若为 Integration: kill_integration_stream_job() 每个 stream
      → GenericJob.enqueue_cancel_pipeline_run()（异步兜底）
```

**注意**：当 `status != FAILED` 时（如超时状态被配置为其他值），**不发送通知**，但仍会取消作业。

### 9.9 多失败块的取值规则

#### failed_block_runs 的范围

`failed_block_runs` 属性（[schedules.py:853](mage_ai/orchestration/db/models/schedules.py#L853)）仅包含状态为 **FAILED** 的块：

```python
def failed_block_runs(self) -> List['BlockRun']:
    return [b for b in self.block_runs if b.status == BlockRun.BlockRunStatus.FAILED]
```

**不包含** UPSTREAM_FAILED、CONDITION_FAILED、CANCELLED 状态的块。

#### error_msg 中的块列表

所有失败块的 UUID 都会被列出，用逗号分隔：

```python
error_msg = 'Failed blocks: ' f'{", ".join([b.block_uuid for b in failed_block_runs])}.'
```

#### stacktrace 的选取规则

`on_pipeline_run_failure()` 中（[pipeline_scheduler_original.py:350](mage_ai/orchestration/pipeline_scheduler_original.py#L350)）选取 stacktrace 的逻辑：

```python
stacktrace = None
for br in failed_block_runs:
    if br.metrics:
        message = br.metrics.get('error', {}).get('message')
        if message:
            message_split = message.split('\n')
            if len(message_split) > 50:
                message_split = message_split[-50:]
                message_split.insert(0, '... (error truncated)')
            message = '\n'.join(message_split)
            stacktrace = f'Error for block {br.block_uuid}:\n{message}'
            break    # ← 第一个有 error.message 的块，break 退出
```

**规则**：
- 按 `failed_block_runs` 的迭代顺序（即 `self.block_runs` 的自然顺序，通常是 DB 主键顺序）
- 取 **第一个** 具有 `metrics.error.message` 的失败块
- 错误消息截断到最后 **50 行**，超过则在开头插入 `'... (error truncated)'`
- 格式：`'Error for block {block_uuid}:\n{truncated_message}'`
- 如果所有失败块都没有 error message，则 `stacktrace = None`

### 9.10 通知发送机制

#### NotificationSender

`NotificationSender`（[sender.py:45](mage_ai/orchestration/notification/sender.py#L45)）支持多种通知渠道：

| 渠道 | 发送函数 |
|---|---|
| Slack | `send_slack_message()` |
| Microsoft Teams | `send_teams_message()` |
| Discord | `send_discord_message()` |
| Google Chat | `send_google_chat_message()` |
| Email | `send_email()` |
| Opsgenie | `send_opsgenie_alert()` |
| Telegram | `send_telegram_message()` |

#### 消息优先级

`__send_pipeline_run_message()`（[sender.py:187](mage_ai/orchestration/notification/sender.py#L187)）的消息构建优先级：

1. **最高**：直接传入的 `summary` 参数（如 `memory_usage_failure` 传入的 summary）
2. **次之**：用户配置的 `message_template`（NotificationConfig 中的自定义模板）
3. **默认**：`DEFAULT_MESSAGES['failure']` 中的默认模板

默认失败消息模板：
- **title**：`'Failed to run Mage pipeline {pipeline_uuid}'`
- **summary**：`'Failed to run Pipeline {pipeline_uuid} with Trigger {pipeline_schedule_id} {pipeline_schedule_name} at execution time {execution_time}. Error: {error}'`

变量插值（`__interpolate_vars`）支持：`error`、`stacktrace`、`execution_time`、`pipeline_run_url`、`pipeline_schedule_id`、`pipeline_schedule_name`、`pipeline_schedule_description`、`pipeline_uuid`。

#### 发送条件

只有当 `AlertOn.PIPELINE_RUN_FAILURE` 在 `config.alert_on` 列表中时，才会实际发送通知。

### 9.11 重试机制

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
   → 异常捕获: 初始化失败 → 直接 send_pipeline_run_failure_message (入口1)
   → PipelineRun.create_block_runs()      # 为所有可执行块创建 BlockRun
2. PipelineScheduler.schedule()
   → update_block_run_statuses()          # 传播 UPSTREAM_FAILED / CONDITION_FAILED
   → 判定管道状态（按优先级）:
     a. all_blocks_completed(allow_blocks_to_fail)
        ├── any_blocks_failed → on_pipeline_run_failure (入口2)
        └── 全部成功 → complete() + 成功通知
     b. __check_pipeline_run_timeout() → on_pipeline_run_failure (入口3)
     c. any_blocks_failed and not allow_blocks_to_fail
        → on_pipeline_run_failure (入口4, 立即失败不等全部完成)
   → executable_block_runs()              # 基于 DB 状态筛选可执行块
   → job_manager.add_job(run_block, ...)  # 提交作业
3. 块完成回调: on_block_complete() → schedule()
   块失败回调: on_block_failure() → 更新 FAILED 状态（不直接触发管道失败）
4. 内存超限: memory_usage_failure() → stop() + 直接发通知 (入口5)
5. on_pipeline_run_failure() 统一处理:
   → UsageStatisticLogger.pipeline_run_ended_sync()
   → status == FAILED 时: 收集 stacktrace + send_pipeline_run_failure_message()
   → cancel_block_runs_and_jobs()         # 批量取消剩余作业
```

---

## 12. 潜在风险

| 风险类别 | 描述 | 来源代码位置 | 严重程度 |
|---|---|---|---|
| **缺失节点静默忽略** | YAML 中引用不存在的块 UUID 不会报错，仅被列表推导式过滤，上游丢失使块意外成为 root block | [pipeline.py:1032](mage_ai/data_preparation/models/pipeline.py#L1032) | 高 |
| **get_block() 仅 print 不抛异常** | 找不到块时 `print` 到 stdout 后返回 `None`，调用方若不检查则后续 AttributeError | [pipeline.py:1913](mage_ai/data_preparation/models/pipeline.py#L1913) | 高 |
| **初始化失败缺失清理（入口1）** | `start()` 初始化异常是**唯一不执行任何清理**的失败路径：无 UsageStatisticLogger、无 cancel_block_runs、无作业 kill。集成管道若在 initialize_state_and_runs() 中途抛异常，可能残留 state/output 文件 | [pipeline_scheduler_original.py:176](mage_ai/orchestration/pipeline_scheduler_original.py#L176) | 中 |
| **stacktrace 只取第一个失败块** | 多失败块场景下通知仅包含第一个有 `error.message` 的块的堆栈，其余失败原因被隐藏，排障困难 | [pipeline_scheduler_original.py:350](mage_ai/orchestration/pipeline_scheduler_original.py#L350) | 中 |
| **stop_pipeline_run 状态守卫陷阱** | `stop_pipeline_run()` 开头检查 `status not in [INITIAL, RUNNING]` 直接 return。若代码路径中先 `update(status=FAILED)` 再调用 `stop()`，清理逻辑会被静默跳过。当前5个入口恰好避开，但后续代码变更易踩坑 | [pipeline_scheduler_original.py:1445](mage_ai/orchestration/pipeline_scheduler_original.py#L1445) | 中 |
| **管道最终状态语义不一致** | 5 个失败入口产生 3 种管道最终状态：入口1/2/4 → FAILED，入口3 → timeout_status（默认 FAILED），入口5 → CANCELLED。下游消费方需同时处理多种失败态，且 CANCELLED 与 FAILED 的区分语义不明显 | [pipeline_scheduler_original.py:629](mage_ai/orchestration/pipeline_scheduler_original.py#L629)、[pipeline_scheduler_original.py:1452](mage_ai/orchestration/pipeline_scheduler_original.py#L1452) | 中 |
| **通知内容策略不统一** | A 类入口传 `error+stacktrace`，B 类（入口5）传 `summary`，C 类（入口1）仅传固定 `error`。消费方（如 Slack hook）需同时适配三种参数组合 | [sender.py:187](mage_ai/orchestration/notification/sender.py#L187) | 低 |
| **图结构修改无事务** | `add_block()` / `update_block()` 中先修改内存结构再 validate，环检测失败时内存状态已被修改 | [pipeline.py:1878](mage_ai/data_preparation/models/pipeline.py#L1878)、[pipeline.py:2137](mage_ai/data_preparation/models/pipeline.py#L2137) | 中 |
| **运行时环检测滞后** | `run_blocks()` 中通过 tries >= 1000 检测环，最多需要 1000 次无效迭代才能发现，且异常消息不含环路径 | [block/\_\_init\_\_.py:242](mage_ai/data_preparation/models/block/__init__.py#L242) | 中 |
| **BlockRun 永久 INITIAL** | 调度器中 `block is None` 的 BlockRun 永远不执行也不报错，最终残留为 INITIAL | [schedules.py:1176](mage_ai/orchestration/db/models/schedules.py#L1176) | 中 |
| **any_blocks_failed 仅统计 FAILED** | 仅检查 FAILED 状态，UPSTREAM_FAILED 块不计入失败统计（虽不影响最终判定，因其上游必有 FAILED） | [schedules.py:1519](mage_ai/orchestration/db/models/schedules.py#L1519) | 低 |
| **动态块 UUID 冲突** | 动态子块使用 `parent_uuid:index` 格式，`get_block()` 用 `split(':')[0]` 回退查找，若用户块 UUID 包含 `:` 可能导致误匹配 | [pipeline.py:1910](mage_ai/data_preparation/models/pipeline.py#L1910) | 低 |
| **Streaming 管道结构限制** | 强制单 source、单上游 transformer 的约束过于严格，无法表达更复杂的 DAG | [streaming_pipeline_executor.py:35](mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L35) | 低 |

---

## 13. 后续研究方向

1. **缺失节点校验增强**：在 `__initialize_blocks_by_uuid()` 中对缺失的引用产生警告日志（而非静默跳过），帮助用户及早发现配置错误

2. **get_block() 异常化**：将 `print` 改为 `logger.error()` 并在关键调用路径中检查返回值，或提供 `get_block_strict()` 变体在找不到块时抛异常

3. **失败入口清理路径统一化**：将入口 1（初始化失败）改为调用统一的 `stop_pipeline_run()` 或 `on_pipeline_run_failure()`，确保 `UsageStatisticLogger`、`cancel_block_runs_and_jobs()` 等清理动作完整执行。考虑抽取 `PipelineFailureHandler` 统一处理所有失败入口

4. **stop_pipeline_run 守卫策略重构**：将"状态不在 INITIAL/RUNNING 直接 return"改为"根据当前状态判定是否需要执行剩余清理步骤"，避免因调用顺序变化导致清理被静默跳过

5. **管道最终状态语义统一**：明确 FAILED / CANCELLED 两种终止状态的语义边界，或统一为单终止态（如 FAILED）+ 子字段（failure_reason），简化下游消费方逻辑

6. **多失败块 stacktrace 聚合**：当前仅取第一个失败块的错误消息，可改为聚合所有失败块的关键信息（如 `block1: KeyError x; block2: Timeout after 300s`），便于一次性排查

7. **增量环检测**：当前 `validate()` 每次全量遍历，对于大型管道可考虑增量检测算法，仅在修改影响范围内检查

8. **图结构修改事务化**：将 `validate()` → `save()` 的过程包装为原子操作，验证失败时回滚内存状态（深拷贝 → 修改 → validate → 赋值）

9. **BFS 环检测优化**：`run_blocks()` 中的 tries >= 1000 兜底可替换为显式入度计数，在 O(V+E) 时间内判断是否有环，并输出具体环路径

10. **BlockRun 超时清理**：对调度器中长期处于 INITIAL 状态的 BlockRun 增加 gc 逻辑或告警，避免因缺失节点导致管道永远无法完成

11. **并行度自适应**：当前 BFS 遍历中同一层的块全部并行执行，可考虑根据系统资源（CPU/内存）和 `concurrency_config.block_run_limit` 动态控制并行度

12. **DAG 可视化与调试**：利用 [presenters/blocks/graph.py](mage_ai/presenters/blocks/graph.py) 中已有的图构建逻辑，增强运行时 DAG 可视化，展示动态块展开后的完整执行图

13. **拓扑排序缓存**：对于结构不变的管道，可缓存拓扑排序结果，避免每次执行都重新遍历

14. **Streaming 管道的 DAG 扩展**：放宽单 source 限制，支持多源流式 DAG，适应更复杂的数据流场景

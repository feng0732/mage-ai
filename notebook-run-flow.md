# Mage AI Notebook 风格运行 - 完整链路协作分析

## 核心结论（先看这里）

Notebook 风格运行有 **三条独立但协作的链路**，分工明确：

| 链路 | 作用 | 通道类型 | 方向 | 核心文件 |
|------|------|----------|------|----------|
| **执行收发通道** | 前端发送「运行 Block」请求 + 接收内核执行结果 | **WebSocket 双向** | 前端 ↔ 后端 | [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py) |
| **SSE 事件流通道** | 测试页等特殊场景的代码执行与结果推送 | SSE (Server-Sent Events) | 后端 → 前端 | [events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py) |
| **变量持久化通道** | Block 输出写入磁盘，供下游 Block 和输出展示读取 | 本地文件系统 | 内核 → 磁盘 → 内核 | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py) |

**⚠️ 关键区分**：Pipeline 编辑页的 Block 运行走 **WebSocket 双向收发**（发请求 + 收结果），测试页走 **SSE 事件流**（发请求用 REST API + 收结果用 SSE）。这两套通道是独立的，不要混淆。

### 变量持久化的三条写入路径（互斥，每次只走一条）

| 路径 | 触发条件 | 写入位置 | 变量名格式 |
|------|----------|----------|------------|
| **普通返回值** | 函数返回普通值（非生成器） | `execute_sync` 外层 | `output_0`, `output_1` |
| **生成器分批** | `MEMORY_MANAGER_V2=True` + 生成器函数 | `execute_block_function` 内部遍历生成器时 | `output_0/0`, `output_0/1`... |
| **PySpark 本地补写** | PySpark 内核 + DATA_LOADER/TRANSFORMER | `%%local` 注入代码，直接调用 | `output_0`, `output_1` |

### 变量读取的三个时机

| 读取时机 | 触发位置 | 目的 | 数据来源 |
|----------|----------|------|----------|
| **输入数据读** | `__get_outputs_from_input_vars()` | 读取上游输出作为输入 | 上游 Block variables 目录 |
| **动态 child 展示读** | `run_task()` → `block.get_outputs()` | 动态 child block 输出展示 | 本 Block variables 目录 |
| **普通 block 展示读** | `__custom_output()` → `block.get_outputs()` | 普通 block 输出展示 | 本 Block variables 目录 |

### 三条链路协作时序（普通返回值路径 - Pipeline 编辑页）

```
前端                          后端                              Jupyter 内核                        磁盘
 │                             │                                 │                                   │
 │ 1. WebSocket: 运行 Block    │                                 │                                   │
 │────────────────────────────>│ 2. 三次代码注入                │                                   │
 │                             │    add_execution_code()        │                                   │
 │                             │    add_internal_output_info()  │                                   │
 │                             │    get_block_output_process_code()（PySpark）│                     │
 │                             │                                 │                                   │
 │                             │ 3. client.execute(code)        │                                   │
 │                             │────────────────────────────────>│ 4. exec(完整代码)                │
 │                             │                                 │    └─ execute_custom_code()      │
 │                             │                                 │       ├─ block = pipeline.get_block()                          │
 │                             │                                 │       ├─ run_upstream_blocks() ← 可选                           │
 │                             │                                 │       └─ block.execute_with_callback()                          │
 │                             │                                 │           └─ execute_sync()    │
 │                             │                                 │               ├─ _store_variables_in_block_function = __store_variables （闭包）
 │                             │                                 │               ├─ execute_block()│
 │                             │                                 │               │   ├─ __get_outputs_from_input_vars()           │
 │                             │                                 │               │   │   └─ fetch_input_variables() ←───── 读取①：输入读
 │                             │                                 │               │   └─ _execute_block()
 │                             │                                 │               │       ├─ exec(self.content, results)
 │                             │                                 │               │       └─ execute_block_function()
 │                             │                                 │               │           ├─ output = block_function()
 │                             │                                 │               │           └─ 生成器路径：遍历分批写入 ──┐
 │                             │                                 │               │              普通路径：仅返回 output    │
 │                             │                                 │               │                                              │
 │                             │                                 │               └─ 普通路径：_store_variables_in_block_function() → 写入①
 │                             │                                 │                  生成器路径：已设为 None，跳过              │
 │                             │                                 │                                                               │
 │                             │                                 │       ├─ block.run_tests()                                     │
 │                             │                                 │       └─ return output                                         │
 │                             │                                 │                                                               │
 │                             │                                 │    └─ __custom_output()   ← 第2次注入的代码                    │
 │                             │                                 │       ├─ if is_dynamic_child: return                           │
 │                             │                                 │       └─ block.get_outputs() ←───── 读取②：普通展示读
 │                             │                                 │          └─ print(render_output_tags(json))                   │
 │                             │                                 │                                                               │
 │                             │ 5. iopub 消息 → parse_output_message()                        │
 │                             │    → WebSocketServer.send_message()                           │
 │                             │◄────────────────────────────────│                               │
 │                             │                                 │                               │
 │ 6. WebSocket onMessage:     │                                 │                               │
 │    解析内核输出并显示        │                                 │                               │
 │◄────────────────────────────│                                 │                               │
```

---

## 一、执行收发通道（WebSocket 双向）

Pipeline 编辑页的 Block 运行使用 **WebSocket 双向收发**：前端通过同一个 WebSocket 连接发请求、收结果。

### 1.1 前端发起请求

用户点击「运行 Block」按钮，前端通过 WebSocket 发送消息：

```json
{
  "pipeline_uuid": "my_pipeline",
  "uuid": "block_uuid",
  "type": "data_loader",
  "code": "@data_loader\ndef load_data():\n    return pd.DataFrame(...)",
  "run_upstream": false,
  "run_tests": true,
  "global_vars": {...}
}
```

### 1.2 后端接收入口

`WebSocketServer.on_message()` 是入口 [websocket_server.py#L185-L310](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py#L185-L310)：

```python
async def on_message(self, raw_message):
    message = json.loads(raw_message)
    
    if message.get('execute_pipeline'):
        self.__execute_pipeline(...)
    else:
        await self.__execute_block(message, pipeline, kernel_name, global_vars)
```

### 1.3 `__execute_block` 核心逻辑

[websocket_server.py#L401-L562](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py#L401-L562)

```python
async def __execute_block(self, message, pipeline, kernel_name, global_vars):
    block_uuid = message.get('uuid')
    custom_code = message.get('code')
    
    block = pipeline.get_block(block_uuid, ...)
    
    # 第1次注入：执行代码包装（execute_custom_code）
    if block.type in CUSTOM_EXECUTION_BLOCK_TYPES:
        code = add_execution_code(
            pipeline_uuid, block_uuid, custom_code, global_vars, ...
        )
    
    # 第2次注入：内部输出信息包装（__custom_output）
    code = add_internal_output_info(block, code, ...)
    
    # 发送到内核执行
    client = self.init_kernel_client(kernel_name)
    msg_id = client.execute(code)
    
    # 第3次注入（可选，PySpark）：输出处理代码（%%local）
    block_output_process_code = get_block_output_process_code(...)
    if block_output_process_code:
        client.execute(block_output_process_code)
```

### 1.4 三次代码注入详解

三次代码注入都发生在后端，在内核执行前，目的是把用户代码包装成完整的可执行脚本。

#### 第1次注入：`add_execution_code()`

[output_display.py#L209-L300](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py#L209-L300)

将 `execute_custom_code.py` 模板注入，替换占位符后生成：

```python
import datetime

def execute_custom_code():
    block_uuid = 'load_data'
    pipeline = Pipeline(uuid='my_pipeline', repo_path='...')
    block = pipeline.get_block('load_data')
    
    code = '''@data_loader
def load_data():
    return pd.DataFrame(...)'''
    
    global_vars = {...}
    
    if run_upstream:
        block.run_upstream_blocks(...)
    
    # 核心：调用 block 的执行方法
    block_output = block.execute_with_callback(
        custom_code=code,
        from_notebook=True,
        global_vars=global_vars,
        ...
    )
    
    if run_tests:
        block.run_tests(...)
    
    output = block_output['output'] or []
    return output

df = execute_custom_code()
```

#### 第2次注入：`add_internal_output_info()`

[output_display.py#L96-L187](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py#L96-L187)

在用户代码最后添加 `__custom_output()` 调用，用于格式化和打印输出：

```python
# 用户代码
@data_loader
def load_data():
    return pd.DataFrame({'a': [1, 2, 3]})

# 注入的后处理代码
__custom_output()  # 调用这个函数格式化并打印输出
```

`__custom_output()` 定义在 [custom_output.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/custom_output.py)，负责：
1. 如果是 dynamic child，直接返回（已在 run_task 中打印）
2. 调用 `block.get_outputs()` 从磁盘读取持久化的变量
3. 根据变量类型（DataFrame、Series、模型等）做不同的格式化
4. 用 `print(render_output_tags(json_string))` 输出，供前端解析

#### 第3次注入：`get_block_output_process_code()`（PySpark 专用）

[output_display.py#L303-L331](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py#L303-L331)

仅用于 PySpark 内核，在 Spark 端执行完后，在本地端存储变量：

```python
%%local
from mage_ai.data_preparation.models.pipeline import Pipeline

block_uuid='load_data'
pipeline = Pipeline(uuid='my_pipeline', repo_path='...')
block = pipeline.get_block(block_uuid)
variable_mapping = dict(df=df)
block.store_variables(variable_mapping)  # 存储到本地磁盘
block.analyze_outputs(variable_mapping)
block.update_status(BlockStatus.EXECUTED)
```

---

## 二、内核执行流程

### 2.1 `execute_with_callback()` - Notebook 执行入口

[block/__init__.py#L1349-L1434](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1349-L1434)

这是 **Notebook 场景专用** 的执行方法，与 Pipeline 调度的 `BlockExecutor.execute()` 不同：

```python
def execute_with_callback(self, global_vars=None, logger=None, **kwargs):
    # 1. 执行条件块（如果有）
    if self.conditional_blocks:
        for conditional_block in self.conditional_blocks:
            result = conditional_block.execute_conditional(...)
    
    try:
        # 2. 核心执行
        output = self.execute_sync(
            global_vars=global_vars,
            logger=logger,
            from_notebook=True,  # 标记来自 Notebook
            **kwargs,
        )
    except Exception as error:
        # 3. 失败回调
        for callback_block in callback_arr:
            callback_block.execute_callback('on_failure', ...)
        raise
    
    # 4. 成功回调
    for callback_block in callback_arr:
        callback_block.execute_callback('on_success', ...)
    
    return output
```

### 2.2 `execute_sync()` - 同步执行（关键：闭包定义 + 两次写入位置）

[block/__init__.py#L1436-L1673](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1436-L1673)

```python
def execute_sync(self, store_variables=True, analyze_outputs=False, ...):
    def __execute(...):
        # Step 1: 定义存储变量的闭包，绑定上下文参数
        def __store_variables(variable_mapping, ...):
            return self.store_variables(
                variable_mapping,
                execution_partition=execution_partition,
                dynamic_block_index=dynamic_block_index,
                ...
            )
        
        self._store_variables_in_block_function = __store_variables  ← 闭包赋值
        
        # Step 2: 执行 Block
        output = self.execute_block(...)
        
        # Step 3: 后处理输出，构建 variable_mapping
        block_output = self.post_process_output(output)
        variable_keys = [f'output_{idx}' for idx in range(len(block_output))]
        variable_mapping = dict(zip(variable_keys, block_output))
        
        # Step 4: 存储变量（外层写入）
        if store_variables and self.pipeline and self.pipeline.type != PipelineType.INTEGRATION:
            # ⚠️ 关键判断：如果 _store_variables_in_block_function 为 None，则跳过
            if self._store_variables_in_block_function and isinstance(variable_mapping, dict):
                self._store_variables_in_block_function(variable_mapping)  ← 写入
        
        # Step 5: 分析输出
        if analyze_outputs:
            self.analyze_outputs(variable_mapping, ...)
        else:
            self.analyze_outputs(variable_mapping, shape_only=True)
        
        return output
```

**关键点**：
- `_store_variables_in_block_function` 在 `execute_sync` 开头被赋值为闭包
- 外层写入在 `execute_block()` 返回之后
- 如果 `_store_variables_in_block_function` 被设为 `None`，外层写入会跳过（生成器路径就是这样）

### 2.3 `execute_block()` - Block 执行核心

[block/__init__.py#L1855-L1931](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1855-L1931)

```python
def execute_block(self, **kwargs):
    with self._redirect_streams(...):
        # 第一步：获取上游输入数据（读取①：输入读）
        (
            outputs_from_input_vars,  # 注入到 exec 的字典，包含 df_1, df_2 等
            input_vars,               # 传递给主函数的参数列表
            kwargs_vars,              # 关键字参数字典
            upstream_block_uuids,     # 上游 Block UUID 列表
        ) = self.__get_outputs_from_input_vars(**kwargs)
        
        # 第二步：执行 Block 代码
        outputs = self._execute_block(
            outputs_from_input_vars,
            **kwargs,
        )
    
    return dict(output=outputs)
```

### 2.4 `_execute_block()` - 实际代码执行

[block/__init__.py#L1966-L2098](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1966-L2098)

```python
def _execute_block(self, outputs_from_input_vars, **kwargs):
    # 合并字典：装饰器函数 + 上游输入变量
    results = merge_dict(
        {
            'preprocesser': self._block_decorator(...),
            'test': self._block_decorator(...),
            self.type: self._block_decorator(...),  # @data_loader 等
        },
        outputs_from_input_vars,  # 包含 df_1, df_2, upstream_block_uuid 等
    )
    
    # 执行用户代码
    if custom_code:
        exec(custom_code, results)  # results 作为全局命名空间
    elif self.content:
        exec(self.content, results)
    
    # 提取装饰器标记的函数
    # 执行 preprocessor 函数
    # 执行主函数 block_function
    outputs = self.execute_block_function(block_function, input_vars, ...)
    
    return outputs
```

**关键点**：`outputs_from_input_vars` 字典被合并到 `results`，作为 `exec` 的全局命名空间，所以用户代码中可以直接使用 `df_1`, `df_2` 等变量。

### 2.5 `execute_block_function()` - 执行主函数（生成器分批写入在这里）

[block/__init__.py#L2100-L2246](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2100-L2246)

```python
def execute_block_function(self, block_function, input_vars, **kwargs):
    # 调用主函数
    if has_kwargs and global_vars:
        output = block_function_updated(*input_vars, **global_vars)
    else:
        output = block_function_updated(*input_vars)
    
    # ⚠️ 只有生成器函数才会在这里写入
    if MEMORY_MANAGER_V2 and inspect.isgeneratorfunction(block_function_updated):
        # 生成器路径：遍历并分批写入
        for data in output:
            variable_mapping = {...}
            self._store_variables_in_block_function(
                variable_mapping,
                clean_variable_uuid=False,
                skip_delete=True,
                ...
            )
        
        # 最后再存一次变量类型
        self._store_variables_in_block_function(
            {'output_0': variable_types},
            save_variable_types_only=True,
        )
        
        # ⚠️ 关键：设为 None，阻止外层再次写入
        self._store_variables_in_block_function = None
    
    if output is None:
        return []
    return output
```

**关键结论**：
- **普通返回值**：`execute_block_function` 只返回 `output`，**不做任何写入**
- **生成器**：`execute_block_function` 内部遍历生成器，**分批写入**，最后设为 `None` 阻止外层重复写入

---

## 三、变量持久化通道 - 三条写入路径完整区分

`_store_variables_in_block_function` 是关键枢纽。它在 `execute_sync` 中被定义为闭包，绑定了执行上下文参数。

### 3.1 路径1：普通返回值写入（同步批量写入）

**触发条件**：函数返回普通值（非生成器），且 `store_variables=True`

**调用位置**：`execute_sync()` 内，`execute_block()` 返回之后 [block/__init__.py#L1630-L1633](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1630-L1633)

**调用链**：
```
execute_sync()
  ├─ _store_variables_in_block_function = __store_variables  ← 闭包赋值
  ├─ execute_block()
  │   └─ execute_block_function()
  │       ├─ output = block_function(*input_vars)  ← 返回普通值
  │       └─ return output  ← 不做任何写入
  ├─ block_output = self.post_process_output(output)
  ├─ variable_mapping = dict(zip(['output_0', 'output_1'], block_output))
  └─ self._store_variables_in_block_function(variable_mapping)  ← 写入在这里
        └─ self.store_variables(variable_mapping, ...)
              └─ variable_manager.add_variable(...)  ← 写入磁盘
```

**变量名格式**：`output_0`, `output_1`, `output_2`...

**写入次数**：1次

### 3.2 路径2：生成器分批写入（流式逐批写入）

**触发条件**：`MEMORY_MANAGER_V2=True` 且 `inspect.isgeneratorfunction(block_function_updated)` [block/__init__.py#L2173](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2173)

**调用位置**：`execute_block_function()` 内部，遍历生成器时 [block/__init__.py#L2193-L2242](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2193-L2242)

**调用链**：
```
execute_sync()
  ├─ _store_variables_in_block_function = __store_variables  ← 闭包赋值
  ├─ execute_block()
  │   └─ execute_block_function()
  │       ├─ output = block_function_updated(*input_vars)  ← 返回生成器对象
  │       └─ if MEMORY_MANAGER_V2 and inspect.isgeneratorfunction(...):
  │             ├─ delete_variables(...)  ← 先清空旧数据
  │             └─ for data in output:     ← 遍历生成器，逐批处理
  │                   ├─ __output_key(0, output_count)
  │                   │     └─ os.path.join(f'output_0', str(output_count))
  │                   │        变量名：output_0/0, output_0/1, output_0/2...
  │                   ├─ variable_mapping = {'output_0/0': data_batch_0}
  │                   ├─ self._store_variables_in_block_function(...)  ← 每批写入一次
  │                   │     └─ variable_manager.add_variable(...)
  │                   └─ output_count += 1
  │             
  │             ├─ # 最后再存一次变量类型
  │             ├─ self._store_variables_in_block_function(
  │                   {'output_0': variable_types},
  │                   save_variable_types_only=True
  │                )
  │             └─ self._store_variables_in_block_function = None  ← 重要！设为 None
  │
  └─ if self._store_variables_in_block_function:  ← 条件为 False，跳过外层写入
         self._store_variables_in_block_function(variable_mapping)
```

**关键特征**：
- 变量名用 `/` 分隔分批：`output_0/0`, `output_0/1`, `output_0/2`...
- 每批数据独立写入磁盘，支持流式处理大数据
- 遍历完成后 `_store_variables_in_block_function` 被设为 `None`，因此 `execute_sync` 外层的写入不会执行

**写入次数**：N次（每批一次）+ 1次（存类型元信息）

### 3.3 路径3：PySpark 本地补写（远程执行后本地补写）

**触发条件**：PySpark 内核 + Block 类型是 `DATA_LOADER` 或 `TRANSFORMER` [output_display.py#L303-L314](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py#L303-L314)

**调用位置**：`get_block_output_process_code()` 返回的代码，通过第三次代码注入，在 Spark 执行完成后，用 `%%local` 在本地端执行

**调用链**：
```
WebSocketServer.__execute_block()
  ├─ code = add_execution_code(...)  ← 第1次注入，代码在 Spark 端执行
  ├─ code = add_internal_output_info(code, ...)  ← 第2次注入
  ├─ client.execute(code)  ← 在 Spark 端执行，结果存在 Spark 集群
  └─ block_output_process_code = get_block_output_process_code(...)  ← 第3次注入
        └─ client.execute(block_output_process_code)  ← 在 %%local 本地端执行
              └─ 执行的代码：
                 %%local
                 from mage_ai.data_preparation.models.pipeline import Pipeline
                 block_uuid='load_data'
                 pipeline = Pipeline(uuid='my_pipeline', repo_path='...')
                 block = pipeline.get_block(block_uuid)
                 variable_mapping = dict(df=df)  # df 从 Spark 拉到本地
                 block.store_variables(variable_mapping)  ← 直接调用，不走闭包
                 block.analyze_outputs(variable_mapping)
                 block.update_status(BlockStatus.EXECUTED)
```

**关键特征**：
- 直接调用 `block.store_variables()`，不走 `_store_variables_in_block_function` 闭包
- 在 `%%local` 模式下执行，运行在本地 Python 进程，不是 Spark 集群
- `df` 变量是通过 Spark magic 的 `-o df` 参数从 Spark 端自动拉取到本地的
- 除了存储变量，还会调用 `analyze_outputs()` 和 `update_status()`

**写入次数**：1次

### 3.4 `store_variables()` - 统一写入入口

[block/__init__.py#L3776-L3861](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3861)

三条路径最终都会调用这个方法：

```python
def store_variables(self, variable_mapping, **kwargs):
    block_uuid, changed = uuid_for_output_variables(...)
    
    if save_variable_types_only:
        # 生成器路径最后保存类型元信息
        for variable_uuid, variable_types in variable_mapping.items():
            self.variable_manager.add_variable_types(...)
        return []
    
    variables_data = self.__store_variables_prepare(variable_mapping, ...)
    
    variables = []
    for uuid, data in variables_data['variable_mapping'].items():
        # PySpark Pipeline 的 pandas DataFrame 自动转 Spark DataFrame
        if spark is not None and self.pipeline.type == PipelineType.PYSPARK and isinstance(data, pd.DataFrame):
            data = spark.createDataFrame(data)
        
        variables.append(
            self.variable_manager.add_variable(
                self.pipeline_uuid, block_uuid, uuid, data, ...
            )
        )
    
    return variables
```

### 3.5 存储目录结构

```
variables/
  └── {pipeline_uuid}/
      └── {block_uuid}/
          ├── output_0/                    # 普通路径 / PySpark 路径
          │   ├── data.parquet
          │   └── metadata.json
          ├── output_0/                    # 生成器路径（分批）
          │   └── 0/
          │       ├── data.parquet
          │       └── metadata.json
          │   └── 1/
          │       ├── data.parquet
          │       └── metadata.json
          ├── output_1/
          │   └── ...
          └── {variable_uuid}/
              └── ...
```

---

## 四、输出展示读取 - 三个读取时机

### 4.1 读取①：输入数据读（上游数据）

**触发位置**：`execute_block()` → `__get_outputs_from_input_vars()` → `fetch_input_variables()`

**时机**：Block 代码执行 **前**

**目的**：读取上游 Block 的输出，作为本 Block 的输入参数

**路径**：上游 Block 的 variables 目录

**代码位置**：[block/__init__.py#L1855-L1860](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1855-L1860)

### 4.2 读取②：动态 child block 展示读

**触发位置**：`run_task()` → `block.get_outputs()`

**时机**：`execute_with_callback()` 完成后，`execute_custom_code()` 内部

**目的**：读取本 Block 刚写入的输出，格式化后打印，供动态 child block 展示

**路径**：本 Block 的 variables 目录

**代码位置**：[execute_custom_code.py#L86-L93](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/execute_custom_code.py#L86-L93)

```python
def run_task(block, execute_kwargs, callback=None, custom_code=None, ...):
    output_dict = block.execute_with_callback(**execute_kwargs)  # 先执行
    
    if callback:
        callback(block)
    
    if run_tests:
        block.run_tests(...)
    
    # 执行完成后，立即读取输出
    outputs = block.get_outputs(dynamic_block_index=dynamic_block_index)
    if outputs is not None and len(outputs) >= 1:
        _json_string = simplejson.dumps(outputs, default=encode_complex, ignore_nan=True)
        return print(render_output_tags(_json_string))  # 打印输出
    
    ...
```

### 4.3 读取③：普通 block 展示读

**触发位置**：`__custom_output()` → `block.get_outputs()`

**时机**：`execute_custom_code()` 返回后，第2次注入的 `__custom_output()` 中

**目的**：读取本 Block 刚写入的输出，格式化后打印，供普通 block 展示

**路径**：本 Block 的 variables 目录

**代码位置**：[custom_output.py#L46-L63](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/custom_output.py#L46-L63)

```python
def __custom_output():
    pipeline = Pipeline.get('{pipeline_uuid}', repo_path='{repo_path}')
    block = pipeline.get_block('{block_uuid}', ...)
    
    if block.executable:
        if is_dynamic_child:
            # 互斥：dynamic child 已在 run_task 中打印，这里直接返回
            return
        
        # 普通 block 走这里
        outputs = block.get_outputs()  ← 从磁盘读取
        
        if outputs is not None and len(outputs) >= 1:
            _json_string = simplejson.dumps(outputs, ...)
            return print(render_output_tags(_json_string))
```

### 4.4 读取②和读取③的互斥关系

**关键**：读取②和读取③是互斥的，每次 Block 运行只走其中一条。

- **动态 child block**：走 `run_tasks()` → `run_task()` → `get_outputs()`（读取②）
- **普通 block**：走 `execute_with_callback()` → 执行完成 → `__custom_output()` → `get_outputs()`（读取③）

`__custom_output()` 中的判断 `if is_dynamic_child: return` 确保了不会重复读取和打印。

### 4.5 完整执行顺序（普通 block）

```
exec(完整代码)
  ├─ execute_custom_code()
  │   ├─ block = pipeline.get_block()
  │   ├─ block.execute_with_callback()
  │   │   └─ execute_sync()
  │   │       ├─ _store_variables_in_block_function = __store_variables  ← 闭包赋值
  │   │       ├─ execute_block()
  │   │       │   ├─ __get_outputs_from_input_vars()
  │   │       │   │   └─ fetch_input_variables() ← 读取①：输入读（上游）
  │   │       │   └─ _execute_block()
  │   │       │       ├─ exec(self.content, results)
  │   │       │       └─ execute_block_function()
  │   │       │           ├─ output = block_function()  ← 执行用户函数
  │   │       │           └─ 生成器路径：遍历并分批写入
  │   │       │              普通路径：仅返回 output
  │   │       └─ 普通路径：_store_variables_in_block_function(variable_mapping) ← 写入
  │   │          生成器路径：_store_variables_in_block_function 已设为 None，跳过
  │   ├─ block.run_tests()
  │   └─ return output
  │
  └─ __custom_output()  ← 第2次注入的代码
      ├─ if is_dynamic_child: return  ← 互斥判断
      ├─ block.get_outputs() ← 读取③：普通展示读
      └─ print(render_output_tags(json))
```

---

## 五、执行结果回传 - 两套独立通道

Mage 有 **两套独立的结果回传通道**，服务于不同页面，不要混淆：

| 通道 | 使用页面 | 请求方式 | 结果回传方式 | 内核类型 |
|------|----------|----------|-------------|----------|
| **WebSocket 收发** | Pipeline 编辑页 | WebSocket `sendMessage` | WebSocket `onMessage` 回调 | Jupyter Kernel (python3/pysparkkernel) |
| **SSE 事件流** | 测试页 `/test` | REST API `code_executions` | SSE `EventSource` | Magic Kernel |

---

### 5.1 通道1：WebSocket 收发（Pipeline 编辑页）

这是 **主要的** Notebook 运行通道。Pipeline 编辑页使用 WebSocket **双向收发**：

**前端代码** [edit.tsx#L2426-L2491](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx#L2426-L2491)：

```typescript
// 前端建立 WebSocket 连接
const { sendMessage } = useWebSocket(getWebSocket(), {
    onMessage: (lastMessage) => {
        // 收到内核输出结果
        const message: KernelOutputType = JSON.parse(lastMessage.data);
        const { block_type, execution_state, msg_type, pipeline_uuid, uuid } = message;
        
        if (msgType !== 'stream_pipeline') {
            // Block 执行结果：设置到对应 block 的 messages 中
            setMessages((prev) => ({
                ...prev,
                [uuid]: messagesFromUUID.concat(message),
            }));
        } else {
            // Pipeline 级别消息
            setPipelineMessages((prev) => [...prev, message]);
        }
        
        // 更新 Block 运行状态
        if (execution_state === 'busy') {
            setRunningBlocks((prev) => prev.concat(block));
        } else if (execution_state === 'idle') {
            setRunningBlocks((prev) => prev.filter(({ uuid: uuid2 }) => uuid !== uuid2));
        }
    },
});
```

**后端结果转发链路**：

```
Jupyter Kernel (子进程)
    │
    │ 执行代码，产生 iopub 消息
    ▼
get_messages()  ← 后台线程持续轮询 [subscriber.py#L8-L24](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/subscriber.py#L8-L24)
    │ client.get_iopub_msg(timeout=1)  ← 从 Jupyter 内核读取 iopub 消息
    │ callback(message)
    ▼
parse_output_message()  ← 解析 Jupyter 消息格式 [kernel_output_parser.py#L25-L86](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/kernel_output_parser.py#L25-L86)
    │ 提取 data/error/execution_state/type
    │ stdout → DataType.TEXT_PLAIN
    │ traceback → DataType.TEXT
    │ execute_result → DataType.TEXT_PLAIN 等
    ▼
WebSocketServer.send_message()  ← 通过 WebSocket 发给所有客户端 [websocket_server.py#L313-L399](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py#L313-L399)
    │ 合并 block_type/block_uuid/pipeline_uuid
    │ client.write_message(json.dumps(message_final))
    ▼
前端 WebSocket onMessage  ← 编辑页接收
```

**关键代码** - 服务器启动时注册回调 [server.py#L745-L749](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/server.py#L745-L749)：

```python
get_messages(
    lambda content: WebSocketServer.send_message(
        parse_output_message(content),
    ),
)
```

这条线在服务器启动时建立，是一个持续运行的后台线程，不断从 Jupyter 内核的 iopub 通道读取消息，解析后通过 WebSocket 推送给前端。

**Jupyter iopub 消息类型**（`parse_output_message` 处理）：

| iopub 消息 | 解析后 DataType | 说明 |
|------------|-----------------|------|
| `stream` (stdout/stderr) | `TEXT_PLAIN` | print 输出 |
| `execute_result` | `TEXT_PLAIN` / `TEXT_HTML` 等 | 最后表达式值 |
| `error` | `TEXT` | 错误信息 + traceback |
| `status` (idle/busy) | 无 data，仅 execution_state | 执行状态 |
| `display_data` (image/png) | `IMAGE_PNG` | 图片输出 |

---

### 5.2 通道2：SSE 事件流（测试页 / Magic Kernel）

测试页 `/test` 使用 **REST API 发请求 + SSE 收结果**，走 Magic Kernel：

**前端代码** [useEventStreams.ts#L48-L78](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/frontend/utils/server/events/useEventStreams.ts#L48-L78)：

```typescript
// 发送代码执行请求（通过 REST API）
const [createMessage, { isLoading }] = useMutation(
    (payload: { message: string }) => {
        return api.code_executions.useCreate()({
            code_execution: {
                message: payload?.message,
                message_request_uuid: getNewUUID(),
                timestamp: Number(new Date()),
                uuid,
            },
        });
    },
);

// 接收结果（通过 SSE EventSource）
eventSourceRef.current = new EventSource(getEventStreamsUrl(uuid));
eventSource.onmessage = (event) => {
    const eventData = JSON.parse(event.data);
    if (eventData.uuid === uuid) {
        setEvents((prev) => [...prev, eventData]);
    }
};
```

**后端结果转发链路**：

```
Magic Kernel 子进程
    │
    │ 执行代码，产生 ExecutionResult
    ▼
read_queue (SyncManager.Queue，跨进程)
    │
    │ ReaderThread 转发
    ▼
write_queue (FasterQueue，主线程)
    │
    │ EventStreamHandler 轮询（每 100ms）
    ▼
SSE 连接 → 前端测试页
```

**EventStreamHandler** [events/stream.py#L20-L63](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py#L20-L63)：

```python
async def get(self, uuid: str) -> None:
    self.set_header('Content-Type', 'text/event-stream')
    
    while True:
        queue = await self.__get_queue()
        result = queue.get_nowait()
        
        if result is not None:
            event_stream = EventStream.load(result=result, uuid=self.uuid, ...)
            self.write(f'data: {event_stream_json}\n\n')
        
        await self.flush()
        await asyncio.sleep(0.1)
```

**Magic Kernel ExecutionResult 类型** [kernels/magic/constants.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/constants.py)：

```python
class ResultType(StrEnum):
    DATA = 'data'      # 最后表达式的值
    STATUS = 'status'  # 状态更新
    STDOUT = 'stdout'  # 标准输出

class ExecutionStatus(StrEnum):
    RUNNING = 'running'
    SUCCESS = 'success'
    ERROR = 'error'
    CANCELLED = 'cancelled'
    READY = 'ready'
```

执行完成时的消息序列：
1. 多个 `STDOUT` - print 输出
2. `STATUS` + `RUNNING` - 代码块执行完成
3. `DATA` + `SUCCESS` - 最后表达式值
4. `STATUS` + `READY` - 内核就绪
5. `None` - 哨兵值

---

### 5.3 两套通道对比

| 维度 | WebSocket 收发 | SSE 事件流 |
|------|---------------|-----------|
| **使用页面** | Pipeline 编辑页（主页面） | 测试页 `/test` |
| **请求方式** | WebSocket `sendMessage` | REST API `code_executions` |
| **结果回传** | WebSocket `onMessage` | SSE `EventSource.onmessage` |
| **内核类型** | Jupyter Kernel (`python3` / `pysparkkernel`) | Magic Kernel（Mage 自研） |
| **消息格式** | Jupyter iopub 消息 → `parse_output_message` 解析 | `ExecutionResult` → `EventStream` 封装 |
| **消息来源** | `get_messages()` 轮询 iopub 通道 | `ReaderThread` 轮询 `read_queue` |
| **后端转发** | `WebSocketServer.send_message()` → `client.write_message()` | `EventStreamHandler` → SSE 推送 |
| **连接管理** | 持久 WebSocket 连接 | SSE 连接 + 自动重连 |
| **服务场景** | 日常开发，Block 编辑与运行 | 内核调试、代码执行测试 |

**⚠️ 不要混淆**：
- Pipeline 编辑页的 Block 运行结果通过 **WebSocket** 回传，不是 SSE
- SSE 只用于测试页的 Magic Kernel 场景
- 两套通道使用不同的内核（Jupyter vs Magic）和不同的消息格式

---

## 六、三条写入路径对比总表

| 维度 | 路径1：普通返回值 | 路径2：生成器分批 | 路径3：PySpark 本地补写 |
|------|-------------------|-------------------|-------------------------|
| **触发条件** | 函数返回普通值 | `MEMORY_MANAGER_V2=True` + 生成器函数 | PySpark 内核 + DATA_LOADER/TRANSFORMER |
| **调用位置** | `execute_sync` 外层（execute_block 返回后） | `execute_block_function` 内部遍历生成器时 | `%%local` 注入代码 |
| **调用入口** | `_store_variables_in_block_function` 闭包 | `_store_variables_in_block_function` 闭包 | 直接 `block.store_variables()` |
| **写入时机** | 函数返回后，一次写入 | 遍历生成器时，每批写入一次 | Spark 执行完后，本地端补写 |
| **变量名格式** | `output_0`, `output_1` | `output_0/0`, `output_0/1`, `output_0/2` | `output_0`, `output_1` |
| **写入次数** | 1次 | N次（每批）+ 1次（存类型） | 1次 |
| **外层是否再写** | 是（就是外层写的） | 否（设为 None 阻止） | 不涉及（不走闭包） |
| **读取展示时机** | `__custom_output()` 中 | `__custom_output()` 中（dynamic child 走 `run_task()`） | `__custom_output()` 中 |
| **典型应用** | 大多数 Block | 大数据流式处理 | PySpark 集群任务 |

---

## 七、关键代码文件索引

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| **WebSocket 入口** | [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py) | 接收前端执行请求，代码注入，提交内核；send_message 推送结果 |
| **iopub 订阅** | [subscriber.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/subscriber.py) | `get_messages()` 持续轮询 Jupyter 内核 iopub 通道 |
| **内核输出解析** | [kernel_output_parser.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/kernel_output_parser.py) | `parse_output_message()` 解析 Jupyter iopub 消息 |
| **内核管理** | [kernels.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/kernels.py) | Jupyter KernelManager 实例（python3 / pysparkkernel） |
| **Active Kernel** | [active_kernel.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/active_kernel.py) | 当前活跃内核的客户端管理 |
| **代码注入** | [output_display.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py) | `add_execution_code`, `add_internal_output_info`, `get_block_output_process_code` |
| **执行代码模板** | [execute_custom_code.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/execute_custom_code.py) | 注入到内核执行的完整脚本，含 `run_task` / `run_tasks` |
| **输出格式化** | [custom_output.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/custom_output.py) | `__custom_output` 函数，格式化输出并打印 |
| **Block 执行** | [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py) | `execute_with_callback`, `execute_sync`, `execute_block`, `_execute_block`, `execute_block_function`, `store_variables`, `get_outputs` |
| **变量持久化** | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py) | `add_variable`, `get_variable`, 磁盘读写 |
| **Magic Kernel** | [kernels/magic/kernels/models.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/kernels/models.py) | Kernel 核心实现，进程池，队列管理 |
| **内核执行** | [kernels/magic/execution.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/execution.py) | `execute_code_async`, `read_stdout_continuously` |
| **结果推送** | [events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py) | SSE 事件流处理器 |
| **队列转发** | [kernels/magic/threads/reader.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/threads/reader.py) | `ReaderThread`, 队列转发 |
| **常量定义** | [kernels/magic/constants.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/constants.py) | `ExecutionStatus`, `ResultType`, `EventStreamType` |

---

## 八、常见混淆点澄清

### Q1: Pipeline 编辑页的 Block 运行结果走 WebSocket 还是 SSE？
- **走 WebSocket**。Pipeline 编辑页使用 WebSocket **双向收发**：发请求用 `sendMessage`，收结果用 `onMessage`
- 后端通过 `get_messages()` → `parse_output_message()` → `WebSocketServer.send_message()` 把 Jupyter 内核的 iopub 消息转发到前端
- SSE 只用于测试页 `/test` 的 Magic Kernel 场景，跟 Pipeline 编辑页无关

### Q2: 变量写入到底在 `execute_block_function` 里面还是外面？
- **普通返回值**：在**外面**（`execute_sync` 外层），`execute_block_function` 只返回 output
- **生成器分批**：在**里面**（`execute_block_function` 内部遍历生成器时），写完还会设为 None 阻止外层重复写
- 不要笼统地说「在里面」或「在外面」，要区分路径

### Q3: `get_outputs` 有几次读取？分别在什么时候？
有 **三次读取**，分工不同：
1. **输入读**：`__get_outputs_from_input_vars()` → 执行前读上游数据
2. **动态 child 展示读**：`run_task()` → `block.get_outputs()` → `execute_with_callback` 完成后立即读
3. **普通 block 展示读**：`__custom_output()` → `block.get_outputs()` → `execute_custom_code()` 返回后读

第2次和第3次是互斥的。动态 child block 走第2次，普通 block 走第3次。

### Q4: 三次代码注入分别干什么？
1. `add_execution_code`：包装成 `execute_custom_code()` 函数调用，内含完整的 Block 执行逻辑
2. `add_internal_output_info`：添加 `__custom_output()` 调用，格式化输出并打印
3. `get_block_output_process_code`：PySpark 专用，`%%local` 模式下本地端存储变量

### Q5: 上游数据怎么传给下游？
- 不是内存传递，是 **磁盘中转**
- 上游：`store_variables()` → 写入磁盘
- 下游：`fetch_input_variables()` → 从磁盘读取
- 变量按 `pipeline/block/variable` 层级存储

### Q6: 生成器路径下为什么 `_store_variables_in_block_function` 要设为 `None`？
- 目的：阻止 `execute_sync` 外层的重复写入
- 机制：生成器在 `execute_block_function` 内部已经分批写入完成了
- 代码：[block/__init__.py#L2242](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2242)
- 外层判断：[block/__init__.py#L1630](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1630) 的 `if self._store_variables_in_block_function` 为 False，跳过

### Q7: `execute_with_callback` 和 `BlockExecutor.execute` 区别？
- `execute_with_callback`：Notebook 交互式运行用，单 Block，带回调
- `BlockExecutor.execute`：Pipeline 调度用，支持重试、状态更新、BlockRun 数据库记录

# Mage AI Notebook 风格运行 - 完整链路协作分析

## 核心结论（先看这里）

Notebook 风格运行有 **三条独立但协作的链路**，分工明确：

| 链路 | 作用 | 通道类型 | 方向 | 核心文件 |
|------|------|----------|------|----------|
| **执行请求通道** | 前端发送「运行 Block」请求，后端接收并提交给内核 | WebSocket | 前端 → 后端 | [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py) |
| **执行结果通道** | 内核执行完代码，把结果推送给前端显示 | SSE (Server-Sent Events) | 后端 → 前端 | [events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py) |
| **变量持久化通道** | Block 输出写入磁盘，供下游 Block 读取 | 本地文件系统 | 内核 → 磁盘 → 内核 | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py) |

三条链路的协作时序：
```
前端                          后端                              内核                           磁盘
 │                             │                                 │                              │
 │ 1. WebSocket: 运行 Block    │                                 │                              │
 │────────────────────────────>│ 2. 代码注入（3次包装）         │                              │
 │                             │    add_execution_code()        │                              │
 │                             │    add_internal_output_info()  │                              │
 │                             │    get_block_output_process_code()                           │
 │                             │                                 │                              │
 │                             │ 3. client.execute(code)        │                              │
 │                             │────────────────────────────────>│ 4. exec(code)                │
 │                             │                                 │    ├─ execute_with_callback()│
 │                             │                                 │    │   └─ execute_sync()     │
 │                             │                                 │    │       └─ execute_block()│
 │                             │                                 │    │           ├─ __get_outputs_from_input_vars()
 │                             │                                 │    │           │   └─ fetch_input_variables() 读取上游
 │                             │                                 │    │           └─ _execute_block()
 │                             │                                 │    │               ├─ exec(self.content, results)
 │                             │                                 │    │               └─ execute_block_function()
 │                             │                                 │    │                   └─ store_variables() ────────────┐
 │                             │                                 │    │                                                         │
 │                             │                                 │    ├─ __custom_output()                                     │
 │                             │                                 │    │   └─ block.get_outputs() ◄──────────────────────────────┘
 │                             │                                 │    └─ print(渲染后的输出)                                   │
 │                             │                                 │                              │                              │
 │                             │ 5. SSE 推送结果               │                              │                              │
 │◄────────────────────────────│◄────────────────────────────────│                              │
 │                             │                                 │                              │
 │ 6. 显示输出                  │                                 │                              │
 │                             │                                 │                              │
 │ 7. 运行下游 Block            │                                 │                              │
 │────────────────────────────>│                                 │                              │
 │                             │                                 │                              │
```

---

## 一、执行请求通道（WebSocket）

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
    
    # 关键：三次代码注入
    if block.type in CUSTOM_EXECUTION_BLOCK_TYPES:
        # 第1次注入：执行代码包装（execute_custom_code）
        code = add_execution_code(
            pipeline_uuid, block_uuid, custom_code, global_vars, ...
        )
    
    # 第2次注入：内部输出信息包装（__custom_output）
    code = add_internal_output_info(block, code, ...)
    
    # 发送到内核执行
    client = self.init_kernel_client(kernel_name)
    msg_id = client.execute(code)
    
    # 第3次注入（可选，PySpark）：输出处理代码
    block_output_process_code = get_block_output_process_code(...)
    if block_output_process_code:
        client.execute(block_output_process_code)
```

### 1.4 三次代码注入详解

这是最容易混淆的地方！**三次代码注入发生在后端，在内核执行前**，目的是把用户代码包装成完整的可执行脚本。

#### 第1次注入：`add_execution_code()`

[output_display.py#L209-L300](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py#L209-L300)

将 `execute_custom_code.py` 模板注入，替换占位符后生成：

```python
# 注入后代码（简化）
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
1. 调用 `block.get_outputs()` 读取持久化的变量
2. 根据变量类型（DataFrame、Series、模型等）做不同的格式化
3. 用 `print(render_output_tags(json_string))` 输出，供前端解析

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

### 2.2 `execute_sync()` - 同步执行

[block/__init__.py#L1436-L1550](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1436-L1550)

核心逻辑：
1. 调用 `execute_block()` 执行代码
2. 如果 `store_variables=True`，调用 `store_variables()` 持久化
3. 如果 `analyze_outputs=True`，调用 `analyze_outputs()` 分析输出
4. 如果 `update_status=True`，更新 Block 状态

### 2.3 `execute_block()` - Block 执行核心

[block/__init__.py#L1855-L1931](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1855-L1931)

```python
def execute_block(self, **kwargs):
    with self._redirect_streams(...):
        # 第一步：获取上游输入数据
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

### 2.5 `execute_block_function()` - 执行主函数并持久化

[block/__init__.py#L2100-L2280](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2100-L2280)

```python
def execute_block_function(self, block_function, input_vars, **kwargs):
    # 调用主函数
    outputs = block_function(*input_vars, **global_vars_or_kwargs)
    
    # 关键：持久化输出变量
    if from_notebook and store_variables:
        # outputs 是函数返回值，可能是 tuple 或单个值
        variable_mapping = self.__build_variable_mapping(
            outputs, 
            dynamic_block_index=dynamic_block_index,
            ...
        )
        # variable_mapping 类似 {'output_0': data1, 'output_1': data2}
        
        self.store_variables(
            variable_mapping,
            execution_partition=execution_partition,
            ...
        )
    
    return outputs
```

---

## 三、变量持久化通道

### 3.1 `store_variables()` - 写入磁盘

[block/__init__.py#L3776-L3861](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3861)

```python
def store_variables(self, variable_mapping, **kwargs):
    # variable_mapping: {'output_0': data1, 'output_1': data2, ...}
    
    variables_data = self.__store_variables_prepare(...)
    
    for uuid, data in variables_data['variable_mapping'].items():
        # 调用 VariableManager 写入磁盘
        self.variable_manager.add_variable(
            self.pipeline_uuid,
            block_uuid,
            uuid,           # 变量名，如 'output_0'
            data,           # 实际数据
            partition=execution_partition,
            ...
        )
    
    return variables
```

### 3.2 `VariableManager.add_variable()` - 实际写入

[variable_manager.py#L61-L150](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py#L61-L150)

```python
def add_variable(self, pipeline_uuid, block_uuid, variable_uuid, data, ...):
    # 推断变量类型
    variable_type, basic_iterable = infer_variable_type(data, ...)
    
    # 创建 Variable 对象
    variable = Variable(
        variable_uuid,
        self.pipeline_path(pipeline_uuid),
        block_uuid,
        variable_type=variable_type,
        ...
    )
    
    # 写入磁盘
    # 路径: {variables_dir}/{pipeline_uuid}/{block_uuid}/{variable_uuid}
    variable.write_data(data)
    
    return variable
```

### 3.3 `get_outputs()` - 读取并格式化输出

[block/__init__.py#L2639-L2725](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2639-L2725)

在 `__custom_output()` 中被调用，用于前端显示：

```python
def get_outputs(self, ...):
    # 从磁盘读取变量
    return get_outputs_for_display_sync(
        self,
        sample=True,
        sample_count=DATAFRAME_SAMPLE_COUNT_PREVIEW,  # 默认 100 行
        ...
    )
```

### 3.4 存储目录结构

```
variables/
  └── {pipeline_uuid}/
      └── {block_uuid}/
          ├── output_0/
          │   ├── data.parquet      # 数据文件
          │   └── metadata.json     # 元信息（类型、shape、sample 等）
          ├── output_1/
          │   └── ...
          └── {variable_uuid}/
              └── ...
```

---

## 四、执行结果通道（SSE 事件流）

### 4.1 Magic Kernel 架构

Magic Kernel 使用三层队列架构，结果通过 SSE 推送到前端：

```
子进程 (执行代码)
    │
    │ 写入 ExecutionResult
    ▼
read_queue (SyncManager.Queue，跨进程)
    │
    │ ReaderThread 转发
    ▼
write_queue (FasterQueue，主线程)
    │
    │ EventStreamHandler 轮询
    ▼
SSE 连接 (前端)
```

### 4.2 `EventStreamHandler` - SSE 服务端

[events/stream.py#L20-L63](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py#L20-L63)

```python
async def get(self, uuid: str) -> None:
    self.set_header('Content-Type', 'text/event-stream')
    self.set_header('Cache-Control', 'no-cache')
    self.set_header('Connection', 'keep-alive')
    
    while True:
        queue = await self.__get_queue()  # 按 uuid 获取 write_queue
        result = None
        
        try:
            result = queue.get_nowait()  # 非阻塞读取
        except Empty:
            pass
        
        if result is not None:
            # 封装为 EventStream
            event_stream = EventStream.load(
                event_uuid=uuid4().hex,
                result=result,
                timestamp=int(datetime.utcnow().timestamp() * 1000),
                type=EventStreamType.EXECUTION,
                uuid=self.uuid,
            )
            
            # SSE 格式推送
            event_stream_json = simplejson.dumps(event_stream, ...)
            self.write(f'data: {event_stream_json}\n\n')
        
        await self.flush()
        await asyncio.sleep(0.1)  # 每 100ms 轮询一次
```

### 4.3 ExecutionResult 类型

[kernels/magic/constants.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/constants.py)

```python
class ResultType(StrEnum):
    DATA = 'data'      # 执行结果数据（最后表达式的值）
    STATUS = 'status'  # 状态更新
    STDOUT = 'stdout'  # 标准输出

class ExecutionStatus(StrEnum):
    RUNNING = 'running'  # 执行中
    SUCCESS = 'success'  # 执行成功
    ERROR = 'error'      # 执行错误
    CANCELLED = 'cancelled'  # 用户取消
    READY = 'ready'      # 内核就绪，可接受新任务
```

执行完成时的消息序列：
1. 多个 `STDOUT` - 执行过程中的 print 输出
2. `STATUS` + `RUNNING` - 代码块执行完成
3. `DATA` + `SUCCESS` - 携带最后表达式的输出值
4. `STATUS` + `READY` - 内核就绪
5. `None` - 哨兵值，标记结束

---

## 五、三条链路协作的完整时序

### 5.1 运行单个 Block 的完整流程

```
时间轴 →
│
1. 前端点击「运行」
   └─ WebSocket 发送消息
      │
      ▼
2. WebSocketServer.on_message() 接收
   └─ __execute_block()
      ├─ add_execution_code()   ← 第1次代码注入
      ├─ add_internal_output_info() ← 第2次代码注入
      ├─ client.execute(code)    ← 提交到内核
      └─ get_block_output_process_code() ← 第3次注入（PySpark）
         │
         ▼
3. 内核进程执行 code
   ├─ exec(完整代码, globals)
   │  └─ execute_custom_code()
   │     ├─ block = pipeline.get_block()
   │     ├─ block.run_upstream_blocks() ← 可选，运行上游
   │     ├─ block.execute_with_callback()
   │     │  ├─ execute_sync()
   │     │  │  └─ execute_block()
   │     │  │     ├─ __get_outputs_from_input_vars()
   │     │  │     │  └─ fetch_input_variables() ← 从磁盘读上游
   │     │  │     └─ _execute_block()
   │     │  │        ├─ exec(self.content, results)
   │     │  │        └─ execute_block_function()
   │     │  │           ├─ outputs = block_function(*input_vars)
   │     │  │           └─ store_variables() ──────┐
   │     │  │                                        ▼
   │     │  │                                     写入磁盘
   │     │  ├─ store_variables() ← 再次持久化（如有）
   │     │  └─ analyze_outputs()
   │     ├─ block.run_tests() ← 运行测试
   │     └─ return output
   │
   └─ __custom_output() ← 第2次注入的代码
      ├─ block.get_outputs() ← 从磁盘读刚才存的
      │  └─ VariableManager.get_variable()
      ├─ format_output_data() ← 格式化（采样、类型转换）
      └─ print(render_output_tags(json))
         │
         ▼
4. AsyncStdout 捕获 print 输出
   └─ read_stdout_continuously 线程
      └─ queue.put(ExecutionResult(type=STDOUT))
         │
         ▼
5. ReaderThread 转发
   ├─ read_queue.get()
   └─ write_queue.put()
      │
      ▼
6. EventStreamHandler 轮询
   ├─ write_queue.get_nowait()
   ├─ 封装为 EventStream
   └─ SSE 推送: data: {...}\n\n
      │
      ▼
7. 前端接收并显示
   ├─ 解析 SSE 消息
   ├─ 区分 STDOUT / DATA / STATUS
   └─ 渲染到 UI
```

### 5.2 关键点：两次读取磁盘

注意这个容易混淆的细节：

1. **第一次读磁盘**：`__get_outputs_from_input_vars()` → `fetch_input_variables()`
   - 时机：Block 执行 **前**
   - 目的：读取上游 Block 的输出，作为本 Block 的输入
   - 路径：上游 Block 的 variables 目录

2. **第二次读磁盘**：`__custom_output()` → `block.get_outputs()`
   - 时机：Block 执行 **后**
   - 目的：读取本 Block 刚写入的输出，格式化后显示给用户
   - 路径：本 Block 的 variables 目录

### 5.3 关键点：两次写入磁盘

同样有两次写入：

1. **第一次写入**：`execute_block_function()` 内部 → `store_variables()`
   - 时机：主函数执行完成后立即
   - 目的：持久化函数返回值，供下游 Block 使用
   - 控制：`store_variables` 参数控制

2. **第二次写入**：`execute_sync()` → `store_variables()`
   - 时机：整个 Block 执行完成后
   - 目的：再次持久化（处理一些边缘情况）
   - 控制：`store_variables` 参数控制

---

## 六、关键代码文件索引

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| **WebSocket 入口** | [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/websocket_server.py) | 接收前端执行请求，代码注入，提交内核 |
| **代码注入** | [output_display.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py) | `add_execution_code`, `add_internal_output_info`, `get_block_output_process_code` |
| **执行代码模板** | [execute_custom_code.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/execute_custom_code.py) | 注入到内核执行的完整脚本 |
| **输出格式化** | [custom_output.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/custom_output.py) | `__custom_output` 函数，格式化输出并打印 |
| **Block 执行** | [block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py) | `execute_with_callback`, `execute_sync`, `execute_block`, `_execute_block`, `execute_block_function`, `store_variables`, `get_outputs` |
| **变量持久化** | [variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py) | `add_variable`, `get_variable`, 磁盘读写 |
| **Magic Kernel** | [kernels/magic/kernels/models.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/kernels/models.py) | Kernel 核心实现，进程池，队列管理 |
| **内核执行** | [kernels/magic/execution.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/execution.py) | `execute_code_async`, `read_stdout_continuously` |
| **结果推送** | [events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py) | SSE 事件流处理器 |
| **队列转发** | [kernels/magic/threads/reader.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/threads/reader.py) | `ReaderThread`, 队列转发 |
| **常量定义** | [kernels/magic/constants.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/constants.py) | `ExecutionStatus`, `ResultType`, `EventStreamType` |

---

## 七、常见混淆点澄清

### Q1: WebSocket 和 SSE 两条通道怎么分工？
- **WebSocket**：双向，前端发请求用（「运行 Block」「取消」等）
- **SSE**：单向，后端推结果用（stdout、执行状态、输出数据）
- WebSocket 也可以推结果，但 Magic Kernel 用 SSE 专门推执行结果，传统 Jupyter Kernel 用 WebSocket 推

### Q2: `store_variables` 和 `get_outputs` 什么关系？
- 时序：先 `store_variables` 写磁盘 → 再 `get_outputs` 读磁盘
- 目的：`store_variables` 为了持久化（给下游 Block 用），`get_outputs` 为了显示（给前端看）
- 位置：都在同一次内核执行中，`store_variables` 在 `execute_block_function` 内部，`get_outputs` 在 `__custom_output` 中

### Q3: 三次代码注入分别干什么？
1. `add_execution_code`：包装成 `execute_custom_code()` 函数调用
2. `add_internal_output_info`：添加 `__custom_output()` 调用，格式化输出
3. `get_block_output_process_code`：PySpark 专用，本地端存储变量

### Q4: 上游数据怎么传给下游？
- 不是内存传递，是 **磁盘中转**
- 上游：`store_variables()` → 写入磁盘
- 下游：`fetch_input_variables()` → 从磁盘读取
- 变量按 `pipeline/block/variable` 层级存储

### Q5: `execute_with_callback` 和 `BlockExecutor.execute` 区别？
- `execute_with_callback`：Notebook 交互式运行用，单 Block，带回调
- `BlockExecutor.execute`：Pipeline 调度用，支持重试、状态更新、BlockRun 数据库记录

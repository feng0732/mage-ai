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

## 三、变量持久化通道 - 三种写入路径完整区分

`_store_variables_in_block_function` 是关键枢纽。它在 `execute_sync` 中被定义为闭包，绑定了执行上下文参数（`execution_partition`, `dynamic_block_index` 等），然后在三个不同路径下被调用。

[block/__init__.py#L1524-L1550](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1524-L1550)

```python
# 在 execute_sync 内部定义的闭包
def __store_variables(variable_mapping, ...):
    return self.store_variables(
        variable_mapping,
        execution_partition=execution_partition,
        dynamic_block_index=dynamic_block_index,
        ...  # 绑定了上下文参数
    )

self._store_variables_in_block_function = __store_variables
```

---

### 3.1 路径1：普通返回值写入（同步批量写入）

**触发条件**：函数返回普通值（非生成器），且 `store_variables=True`

**调用位置**：`execute_sync()` 内，`execute_block()` 返回之后 [block/__init__.py#L1590-L1640](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1590-L1640)

**调用链**：
```
execute_sync()
  └─ execute_block() → execute_block_function()
        ├─ output = block_function(*input_vars)  # 函数返回普通值
        └─ return output
  ├─ block_output = self.post_process_output(output)
  ├─ variable_mapping = dict(zip(['output_0', 'output_1'], block_output))
  └─ self._store_variables_in_block_function(variable_mapping)  ← 这里调用
        └─ self.store_variables(variable_mapping, ...)
              └─ variable_manager.add_variable(...)  ← 写入磁盘
```

**变量名格式**：`output_0`, `output_1`, `output_2`...

**关键代码**：
```python
# execute_sync L1612-L1633
output_count = len(block_output)
variable_keys = [f'output_{idx}' for idx in range(output_count)]
variable_mapping = dict(zip(variable_keys, block_output))

if store_variables and self.pipeline and self.pipeline.type != PipelineType.INTEGRATION:
    if self._store_variables_in_block_function and isinstance(variable_mapping, dict):
        self._store_variables_in_block_function(variable_mapping)
```

---

### 3.2 路径2：生成器分批写入（流式逐批写入）

**触发条件**：`MEMORY_MANAGER_V2=True` 且 `inspect.isgeneratorfunction(block_function_updated)` [block/__init__.py#L2173](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2173)

**调用位置**：`execute_block_function()` 内部，遍历生成器时 [block/__init__.py#L2193-L2242](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2193-L2242)

**调用链**：
```
execute_block_function()
  ├─ output = block_function_updated(*input_vars)  # 返回生成器对象
  └─ if MEMORY_MANAGER_V2 and inspect.isgeneratorfunction(...):
        ├─ delete_variables(...)  ← 先清空旧数据
        └─ for data in output:     ← 遍历生成器，逐批处理
              ├─ __output_key(0, output_count)  ← 生成变量名
              │     └─ os.path.join(f'output_0', str(output_count))
              │        变量名格式：output_0/0, output_0/1, output_0/2...
              ├─ variable_mapping = {'output_0/0': data_batch_0}
              ├─ self._store_variables_in_block_function(
                    variable_mapping,
                    clean_variable_uuid=False,
                    skip_delete=True,
                    override_outputs=False if output_count >= 1 else True
                 )
              │     └─ variable_manager.add_variable(...)  ← 每批写入一次
              └─ output_count += 1
        
        ├─ # 最后再存一次变量类型
        ├─ self._store_variables_in_block_function(
              {'output_0': variable_types},
              save_variable_types_only=True
           )
        └─ self._store_variables_in_block_function = None  ← 重要！设为 None，阻止外层再次写入
```

**关键特征**：
- 变量名用 `/` 分隔分批：`output_0/0`, `output_0/1`, `output_0/2`...
- 每批数据独立写入磁盘，支持流式处理大数据
- 遍历完成后 `_store_variables_in_block_function` 被设为 `None`，因此 `execute_sync` 外层的第二次写入不会执行（L1630 的 `if self._store_variables_in_block_function` 为 False）

---

### 3.3 路径3：PySpark 本地补写（远程执行后本地补写）

**触发条件**：PySpark 内核 + Block 类型是 `DATA_LOADER` 或 `TRANSFORMER` [output_display.py#L303-L314](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/output_display.py#L303-L314)

**调用位置**：`get_block_output_process_code()` 返回的代码，通过第三次代码注入，在 Spark 执行完成后，用 `%%local` 在本地端执行

**调用链**：
```
WebSocketServer.__execute_block()
  ├─ code = add_execution_code(...)  ← 第1次注入，代码会在 Spark 端执行
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
                 variable_mapping = dict(df=df)  # df 是从 Spark 拉到本地的
                 block.store_variables(variable_mapping)  ← 直接调用，不走闭包
                 block.analyze_outputs(variable_mapping)
                 block.update_status(BlockStatus.EXECUTED)
```

**关键特征**：
- 直接调用 `block.store_variables()`，不走 `_store_variables_in_block_function` 闭包
- 在 `%%local` 模式下执行，运行在本地 Python 进程，不是 Spark 集群
- `df` 变量是通过 Spark magic 的 `-o df` 参数从 Spark 端自动拉取到本地的
- 除了存储变量，还会调用 `analyze_outputs()` 和 `update_status()`

---

### 3.4 `store_variables()` - 统一写入入口

[block/__init__.py#L3776-L3861](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L3776-L3861)

三条路径最终都会调用这个方法：

```python
def store_variables(self, variable_mapping, **kwargs):
    block_uuid, changed = uuid_for_output_variables(...)
    
    if save_variable_types_only:
        # 生成器路径最后保存类型元信息
        for variable_uuid, variable_types in variable_mapping.items():
            self.variable_manager.add_variable_types(
                self.pipeline_uuid, block_uuid, variable_uuid, variable_types, ...
            )
        return []
    
    variables_data = self.__store_variables_prepare(variable_mapping, ...)
    
    variables = []
    for uuid, data in variables_data['variable_mapping'].items():
        # PySpark Pipeline 的 pandas DataFrame 自动转 Spark DataFrame
        if spark is not None and self.pipeline.type == PipelineType.PYSPARK and isinstance(data, pd.DataFrame):
            data = spark.createDataFrame(data)
        
        variables.append(
            self.variable_manager.add_variable(
                self.pipeline_uuid,
                block_uuid,
                uuid,
                data,
                ...
            )
        )
    
    return variables
```

### 3.5 `VariableManager.add_variable()` - 实际写入磁盘

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

### 3.6 存储目录结构

```
variables/
  └── {pipeline_uuid}/
      └── {block_uuid}/
          ├── output_0/                    # 普通路径
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

### 3.7 输出展示读取的准确时机

**不是一次读取，是两次读取，分别在不同位置**：

#### 读取1：动态 child block - 在 `run_task()` 中

[execute_custom_code.py#L62-L99](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/execute_custom_code.py#L62-L99)

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
    
    output = []
    if output_dict and output_dict.get('output'):
        output = output_dict.get('output')
    
    return output
```

**调用时机**：在 `execute_custom_code()` 内部，`execute_with_callback()` 完成后立即调用。

**适用场景**：动态 child block（`is_dynamic_child=True`），在 `run_tasks()` 中被循环调用。

#### 读取2：普通 block - 在 `__custom_output()` 中

[custom_output.py#L46-L63](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/custom_output.py#L46-L63)

```python
def __custom_output():
    pipeline = Pipeline.get('{pipeline_uuid}', repo_path='{repo_path}')
    block = pipeline.get_block('{block_uuid}', ...)
    
    if block.executable:
        if is_dynamic_child:
            # dynamic child 的输出已经在 run_task 中打印了，这里直接返回
            return
        
        # 普通 block 走这里
        outputs = block.get_outputs()  ← 从磁盘读取
        
        if outputs is not None and isinstance(outputs, list):
            outputs = outputs[: int('{DATAFRAME_SAMPLE_COUNT_PREVIEW}')]
        
        if outputs is not None and len(outputs) >= 1:
            _json_string = simplejson.dumps(outputs, default=encode_complex, ignore_nan=True)
            return print(render_output_tags(_json_string))  # 打印输出
    
    # ... 后续处理最后一个表达式
```

**调用时机**：`execute_custom_code()` 返回后，第2次代码注入的 `__custom_output()` 被调用。

**适用场景**：普通 block（非动态 child）。

**完整执行顺序**：
```
exec(完整代码)
  ├─ execute_custom_code()
  │   ├─ block.execute_with_callback()
  │   │   └─ execute_sync()
  │   │       ├─ _store_variables_in_block_function = __store_variables  ← 闭包赋值
  │   │       ├─ execute_block()
  │   │       │   └─ execute_block_function()
  │   │       │       ├─ output = block_function()  ← 执行用户函数
  │   │       │       └─ 生成器路径：遍历并分批 store_variables
  │   │       │          普通路径：只返回 output
  │   │       └─ 普通路径：_store_variables_in_block_function(variable_mapping)  ← 写入磁盘
  │   │          生成器路径：_store_variables_in_block_function 已设为 None，跳过
  │   ├─ block.run_tests()
  │   └─ return output
  │
  └─ __custom_output()  ← 第2次注入的代码，在 execute_custom_code() 之后执行
      ├─ block.get_outputs()  ← 从磁盘读取刚才写入的数据
      └─ print(render_output_tags(json))  ← 打印格式化输出
```

---

### 3.8 三种写入路径对比表

| 维度 | 路径1：普通返回值 | 路径2：生成器分批 | 路径3：PySpark 本地补写 |
|------|-------------------|-------------------|-------------------------|
| **触发条件** | 函数返回普通值 | `MEMORY_MANAGER_V2=True` + 生成器函数 | PySpark 内核 + DATA_LOADER/TRANSFORMER |
| **调用位置** | `execute_sync` 外层 | `execute_block_function` 内部 | `%%local` 注入代码 |
| **调用入口** | `_store_variables_in_block_function` | `_store_variables_in_block_function` | 直接 `block.store_variables()` |
| **写入时机** | 函数返回后，一次写入 | 遍历生成器时，每批写入一次 | Spark 执行完后，本地端补写 |
| **变量名格式** | `output_0`, `output_1` | `output_0/0`, `output_0/1`, `output_0/2` | `output_0`, `output_1` |
| **后续外层写入** | 执行 | 不执行（设为 None） | 不涉及 |
| **读取时机** | `__custom_output()` 中 | `__custom_output()` 中（dynamic child 走 `run_task()`） | `__custom_output()` 中 |

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
   │     ├─ is_dynamic_child 判断
   │     │  ├─ 是 → run_tasks() → 循环调用 run_task()
   │     │  │     ├─ block.execute_with_callback(**options)
   │     │  │     ├─ block.run_tests(...)
   │     │  │     └─ outputs = block.get_outputs() ← 读取1：动态 child 展示读
   │     │  │        └─ print(render_output_tags(json))
   │     │  └─ 否 → block.execute_with_callback(**options)
   │     │        ├─ execute_sync()
   │     │        │  ├─ _store_variables_in_block_function = __store_variables ← 闭包赋值
   │     │        │  ├─ execute_block()
   │     │        │  │  ├─ __get_outputs_from_input_vars()
   │     │        │  │  │  └─ fetch_input_variables() ← 读取2：输入数据读（上游）
   │     │        │  │  └─ _execute_block()
   │     │        │  │     ├─ exec(self.content, results)
   │     │        │  │     └─ execute_block_function()
   │     │        │  │        ├─ output = block_function(*input_vars)
   │     │        │  │        └─ 生成器路径：遍历 output，分批 store_variables()
   │     │        │  │              变量名：output_0/0, output_0/1...
   │     │        │  │              最后 _store_variables_in_block_function = None
   │     │        │  └─ 普通路径：_store_variables_in_block_function(variable_mapping) ← 写入1
   │     │        │     生成器路径：已设为 None，跳过
   │     │        └─ analyze_outputs()
   │     ├─ block.run_tests(...) ← 运行测试
   │     └─ return output
   │
   └─ __custom_output() ← 第2次注入的代码
      ├─ if is_dynamic_child: return ← 互斥：动态 child 已在 run_task 打印
      ├─ block.get_outputs() ← 读取3：普通 block 展示读
      │  └─ VariableManager.get_variable()
      ├─ 变量类型判断和格式化
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

### 5.2 关键点：读取磁盘的三个时机

不是两次，是三次读取，分工明确：

1. **第一次读磁盘（输入数据）**：`__get_outputs_from_input_vars()` → `fetch_input_variables()`
   - 时机：Block 代码执行 **前**
   - 目的：读取上游 Block 的输出，作为本 Block 的输入参数
   - 路径：上游 Block 的 variables 目录
   - 代码位置：[block/__init__.py#L1855-L1860](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1855-L1860)

2. **第二次读磁盘（动态 child 输出展示）**：`run_task()` → `block.get_outputs()`
   - 时机：`execute_with_callback()` 完成后，`execute_custom_code()` 内部
   - 目的：读取本 Block 刚写入的输出，格式化后打印，供动态 child block 展示
   - 路径：本 Block 的 variables 目录
   - 代码位置：[execute_custom_code.py#L86-L93](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/execute_custom_code.py#L86-L93)

3. **第三次读磁盘（普通 block 输出展示）**：`__custom_output()` → `block.get_outputs()`
   - 时机：`execute_custom_code()` 返回后，第2次注入的 `__custom_output()` 中
   - 目的：读取本 Block 刚写入的输出，格式化后打印，供普通 block 展示
   - 路径：本 Block 的 variables 目录
   - 代码位置：[custom_output.py#L51-L63](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/utils/custom_output.py#L51-L63)

**关键互斥**：第二次和第三次读取是互斥的。`__custom_output()` 中会判断 `if is_dynamic_child: return`，所以动态 child block 走第二次读取，普通 block 走第三次读取。

---

### 5.3 关键点：写入磁盘的三种路径

不是两次，是三条独立路径，互斥执行（每次 Block 运行只走其中一条）：

| 路径 | 触发条件 | 写入位置 | 写入次数 |
|------|----------|----------|----------|
| **路径1：普通返回值** | 函数返回非生成器 | `execute_sync` 外层 | 1次 |
| **路径2：生成器分批** | MEMORY_MANAGER_V2=True + 生成器函数 | `execute_block_function` 内部遍历生成器时 | N次（每批一次）+ 1次存类型 |
| **路径3：PySpark 本地补写** | PySpark 内核 + DATA_LOADER/TRANSFORMER | `%%local` 注入代码 | 1次 |

**关键互斥**：
- 生成器路径下，`_store_variables_in_block_function` 最后被设为 `None`，因此 `execute_sync` 外层的写入不会执行
- PySpark 路径下，`store_variables` 直接调用，不走 `_store_variables_in_block_function` 闭包
- 普通路径下，只有 `execute_sync` 外层的一次写入

**完整执行顺序对比**：

```
普通返回值路径：
execute_sync()
  ├─ _store_variables_in_block_function = __store_variables
  ├─ execute_block()
  │   └─ execute_block_function()
  │       ├─ output = block_function()  ← 返回普通值
  │       └─ return output  ← 不做任何写入
  └─ _store_variables_in_block_function(variable_mapping)  ← 这里写入一次


生成器分批路径：
execute_sync()
  ├─ _store_variables_in_block_function = __store_variables
  ├─ execute_block()
  │   └─ execute_block_function()
  │       ├─ output = block_function()  ← 返回生成器对象
  │       └─ for data in output:        ← 遍历生成器
  │             └─ _store_variables_in_block_function(...)  ← 每批写入一次
  │       ├─ _store_variables_in_block_function(...)  ← 最后存一次类型
  │       └─ _store_variables_in_block_function = None  ← 设为 None
  └─ if _store_variables_in_block_function:  ← 条件为 False，跳过
         _store_variables_in_block_function(...)


PySpark 本地补写路径：
client.execute(code)  ← Spark 端执行 add_execution_code 注入的代码
  └─ execute_custom_code()
      └─ block.execute_with_callback()
          └─ ...  ← Spark 端执行，数据在 Spark 集群

client.execute(block_output_process_code)  ← 本地端执行 %%local 代码
  └─ block.store_variables(variable_mapping)  ← 直接调用，写入本地磁盘
```

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

### Q2: `store_variables` 有几条调用路径？`get_outputs` 在什么时候读？

**写入有三条互斥路径**：
1. **普通返回值**：`execute_sync` 外层调用 `_store_variables_in_block_function(variable_mapping)`，写入一次
2. **生成器分批**：`execute_block_function` 内部遍历生成器，每批调用一次，最后存一次类型，然后设为 `None` 阻止外层写入
3. **PySpark 本地补写**：`%%local` 注入代码直接调用 `block.store_variables(variable_mapping)`，不走闭包

**读取有三次，分工不同**：
1. **输入读**：`__get_outputs_from_input_vars()` → 执行前读上游数据
2. **动态 child 展示读**：`run_task()` → `block.get_outputs()` → `execute_with_callback` 完成后立即读
3. **普通 block 展示读**：`__custom_output()` → `block.get_outputs()` → `execute_custom_code()` 返回后读

**时序关系**：每次都是先写（三条路径之一）→ 再读（第2或第3次读取）

### Q6: 生成器路径下为什么 `_store_variables_in_block_function` 要设为 `None`？
- 目的：阻止 `execute_sync` 外层的重复写入
- 机制：生成器在 `execute_block_function` 内部已经分批写入完成了
- 代码：[block/__init__.py#L2242](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L2242)
- 外层判断：[block/__init__.py#L1630](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1630) 的 `if self._store_variables_in_block_function` 为 False，跳过

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

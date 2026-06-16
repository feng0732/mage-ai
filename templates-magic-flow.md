# Templates → Magic 执行链：入口衔接、数据交接、状态回传与出错处理

## 一、全局架构：两套内核，一条链路

Mage AI 实际存在 **两套内核**，通过环境变量 `KERNEL_MANAGER` 选择：

| 环境变量值 | 内核 | 代码入口 |
|-----------|------|---------|
| `default`（默认） | **Jupyter Kernel**（ipykernel） | [active_kernel.py](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/active_kernel.py) → `client.execute(code)` |
| `magic` | **Magic Kernel**（自研多进程池） | [CodeExecutionResource.py](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/api/resources/CodeExecutionResource.py) → `KernelManager.get_kernel().run(message)` |

判断逻辑定义在 [server.py#L245](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/settings/server.py#L245)：

```python
KERNEL_MAGIC = os.getenv('KERNEL_MANAGER', 'default') == 'magic'
```

> **本文重点梳理：模板生成代码后，如何通过 Jupyter Kernel（主路径）和 Magic Kernel（新路径）完成执行与回传。**

---

## 二、入口衔接：模板代码如何变成可执行代码

### 2.1 Block 创建时写入模板代码

**调用链**：

```
前端 POST /api/blocks
    │
    ▼
BlockResource.create()
    │
    ▼
Block.create(name, block_type, repo_path, config, language, pipeline)
    │  [block/__init__.py#L1046-L1165]
    │
    ├─► 计算文件路径：{repo_path}/{block_directory}/{uuid}.{ext}
    │     例：/repo/data_loaders/load_users.py
    │
    ├─► 文件不存在 → load_template(block_type, config, file_path, language, pipeline_type)
    │     [template.py#L121-L134]
    │     │
    │     ├─► fetch_template_source() → Jinja2 渲染出代码字符串
    │     └─► write_template(template_source, dest_path) → 写入 .py 文件
    │
    └─► 文件已存在 → 跳过模板写入（保留已有代码）
```

关键代码在 [block/__init__.py#L1145-L1152](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1145-L1152)：

```python
load_template(
    block_type,
    config,
    file_path,
    language=language,
    pipeline_type=pipeline.type if pipeline is not None else None,
)
```

**数据交接点 1**：模板渲染的代码字符串 → `.py` 文件落盘。

### 2.2 Block 执行时读取代码

Block 的 `content` 属性是获取代码的入口，定义在 [block/__init__.py#L499-L510](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L499-L510)：

```python
@property
def content(self) -> str:
    if self.replicated_block and self.replicated_block_object:
        self._content = self.replicated_block_object.content
    if BlockType.GLOBAL_DATA_PRODUCT == self.type:
        return ''
    if self._content is None:
        self._content = self.file.content()   # ← 从磁盘读取 .py 文件
    return self._content
```

> 模板代码写入磁盘后，执行时通过 `block.content` 或前端传入的 `custom_code` 读取。前端可以传 `code` 字段覆盖磁盘代码（用户在编辑器中的实时修改）。

---

## 三、数据交接：代码字符串如何进入执行内核

### 3.1 主路径：WebSocket → Jupyter Kernel

核心逻辑在 [websocket_server.py#L401-L543](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/websocket_server.py#L401-L543) 的 `__execute_block` 方法。

#### 3.1.1 代码预处理三步曲

**第 1 步：获取代码**（[websocket_server.py#L410-L437](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/websocket_server.py#L410-L437)）

```python
custom_code = message.get('code')   # 前端传入的编辑器代码
block = pipeline.get_block(block_uuid)
code = custom_code                   # 默认使用前端传入代码
```

**第 2 步：包装执行框架**（[websocket_server.py#L480-L515](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/websocket_server.py#L480-L515)）

对于 `CUSTOM_EXECUTION_BLOCK_TYPES`（data_loader/transformer/data_exporter/sensor 等），调用 `add_execution_code()`：

```python
code = add_execution_code(
    pipeline_uuid,
    block_uuid,
    custom_code,         # ← 用户代码作为嵌入参数
    global_vars,
    pipeline.repo_path,
    ...
)
```

`add_execution_code`（[output_display.py#L209-L300](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/utils/output_display.py#L209-L300)）将用户代码嵌入模板 [execute_custom_code.py](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/utils/execute_custom_code.py)：

```
最终代码结构：
┌──────────────────────────────────────────┐
│ import datetime                          │
│                                          │
│ {execute_custom_code.py 的全部内容}       │  ← 框架代码
│   - 构建 Pipeline/Block 对象             │
│   - code = r'''{用户代码}'''             │  ← 嵌入的用户代码
│   - block.execute_with_callback(...)     │
│                                          │
│ df = execute_custom_code()               │
└──────────────────────────────────────────┘
```

**第 3 步：追加输出处理**（[websocket_server.py#L524-L531](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/websocket_server.py#L524-L531)）

```python
msg_id = client.execute(
    add_internal_output_info(block, code, ...)
)
```

`add_internal_output_info`（[output_display.py#L96-L187](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/utils/output_display.py#L96-L187)）在代码末尾追加 `custom_output.py` 的内容，用于：
- 捕获最后一条表达式的值
- 将 DataFrame / dict / list 等序列化为 JSON
- 通过 `print()` 输出到 stdout，由 Jupyter kernel 捕获

#### 3.1.2 提交到 Jupyter Kernel

```python
client = self.init_kernel_client(kernel_name)   # 获取 Jupyter KernelClient
msg_id = client.execute(code)                   # 提交执行
WebSocketServer.running_executions_mapping[msg_id] = value  # 记录 block_uuid 映射
```

**数据交接点 2**：预处理后的代码字符串 → `client.execute(code)` → Jupyter Kernel 的执行队列。

### 3.2 新路径：REST API → Magic Kernel

核心逻辑在 [CodeExecutionResource.py](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/api/resources/CodeExecutionResource.py)：

```python
class CodeExecutionResource(GenericResource):
    @classmethod
    async def create(cls, payload, user, **kwargs):
        message = payload.get('message', '')
        uuid = payload.get('uuid')

        kernel = KernelManager.get_kernel(uuid, num_processes=num_processes)
        process = kernel.run(
            message,                          # ← 代码字符串直接传入
            message_request_uuid=message_request_uuid,
            timestamp=now,
        )
        return cls(process, user, **kwargs)
```

**数据交接点 3**：代码字符串 → `kernel.run(message)` → 进程池 `apply_async` → 子进程 `exec(compiled_code, exec_globals)`。

### 3.3 两种路径的数据交接对比

| 维度 | Jupyter Kernel 路径 | Magic Kernel 路径 |
|------|---------------------|-------------------|
| **入口** | WebSocket `on_message` | REST API `POST /api/code_executions` |
| **代码预处理** | `add_execution_code` + `add_internal_output_info`（重包装） | 无预处理，直接执行 |
| **执行方式** | `KernelClient.execute(code)` → ipykernel | `kernel.run(message)` → 进程池 exec |
| **上下文** | Jupyter kernel 维护持久命名空间 | 每次执行独立 `exec_globals = {}` |
| **适用场景** | 交互式 Notebook（主路径） | 隔离执行/并发任务 |

---

## 四、状态回传：执行结果如何回到前端

### 4.1 Jupyter Kernel 路径的状态回传

```
子进程（ipykernel）
    │
    │ stdout / stderr / execute_result / error
    ▼
Jupyter Kernel 的 iopub channel
    │
    │ client.get_iopub_msg(timeout=1)
    ▼
subscriber.py: get_messages(callback)        [server.py#L745-L749]
    │
    │ parse_output_message(message)           [kernel_output_parser.py#L25-L86]
    │ 将 Jupyter 消息格式转为统一 dict:
    │   { data, error, execution_state, msg_id, msg_type, type }
    ▼
WebSocketServer.send_message(message_dict)    [websocket_server.py#L312-L399]
    │
    │ 1. 查 running_executions_mapping[msg_id] 获取 block_uuid/pipeline_uuid
    │ 2. 有 error → format_error() 过滤 Mage 内部堆栈
    │ 3. filter_out_sensitive_data() 过滤环境变量
    │ 4. merge_dict(message, {block_type, pipeline_uuid, uuid})
    ▼
client.write_message(json.dumps(message_final))  ← SSE/WebSocket 推送到浏览器
```

**消息格式解析**（[kernel_output_parser.py#L25-L86](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/kernel_output_parser.py#L25-L86)）：

| Jupyter msg_type | 处理方式 | DataType |
|-----------------|---------|----------|
| `stream`（stdout） | `text.split('\n')` 逐行 | `TEXT_PLAIN` |
| `stream`（stderr） | 同上 | `TEXT_PLAIN` |
| `error` | 提取 `traceback` 列表 | `TEXT` |
| `execute_result` | 取 `data.text/plain` 或 `text/html` | `TEXT_PLAIN` / `TEXT_HTML` |
| `display_data` 含 image | 取 `data.image/png` | `IMAGE_PNG` |
| `execute_input` | 取 `content.code` | `TEXT` |
| `comm_msg`（进度） | 取 widget value | `PROGRESS` |

**超长输出截断**：stdout 超过 `MAX_PRINT_OUTPUT_LINES` 时截断并追加 `... (output truncated)`。

### 4.2 Magic Kernel 路径的状态回传

```
子进程（Pool worker）
    │
    │ execute_code_async() 逐条 put 到 read_queue
    │   - ExecutionResult(STDOUT, RUNNING)   ← 实时 print 输出
    │   - ExecutionResult(DATA, SUCCESS)     ← 最终求值结果
    │   - ExecutionResult(STATUS, READY)     ← 内核就绪
    │   - ExecutionResult(STATUS, ERROR)     ← 异常
    │   - None                               ← 哨兵
    ▼
ReaderThread（主进程 daemon 线程）
    │
    │ read_queue.get() → write_queue.put()
    ▼
全局 execution_result_queue[uuid]      [queues/manager.py]
    │
    │ EventStreamHandler.get(uuid)          [events/stream.py#L28-L59]
    │ SSE (Server-Sent Events) 长轮询：
    │   while True:
    │     queue.get_nowait() → EventStream.load(...) → self.write(f'data: {json}\n\n')
    │     asyncio.sleep(0.1)
    ▼
浏览器 EventSource API 接收
```

**EventStream 消息结构**：

```json
{
  "event_uuid": "...",
  "timestamp": 1718534400000,
  "uuid": "kernel_uuid",
  "type": "execution",
  "result": {
    "data_type": "TEXT_PLAIN",
    "output": "Hello World",
    "process": { "exitcode": null, "is_alive": true, "pid": "...", "uuid": "..." },
    "status": "running",
    "type": "stdout"
  }
}
```

### 4.3 两种回传路径对比

| 维度 | Jupyter Kernel | Magic Kernel |
|------|---------------|--------------|
| **传输协议** | WebSocket（双向） | SSE（单向推送） |
| **推送频率** | 1s 轮询 iopub | 0.1s 轮询队列 |
| **消息映射** | `msg_id → block_uuid`（内存字典） | `uuid → FasterQueue`（全局字典） |
| **错误过滤** | `format_error()` 去除内部堆栈 | `ErrorDetails.from_current_error()` 保留完整堆栈 |
| **敏感数据** | `filter_out_env_var_values()` | 无额外过滤 |

---

## 五、出错处理：各层的错误捕获与回传

### 5.1 Jupyter Kernel 路径的错误处理

#### 5.1.1 代码执行错误

```
用户代码抛异常
    │
    ▼
ipykernel 生成 msg_type='error' 消息，含 traceback 列表
    │
    ▼
kernel_output_parser.py: 提取 traceback → error 字段
    │
    ▼
WebSocketServer.send_message():
    │ error 不为 None → format_error(error, block_uuid)
    │   [websocket_server.py#L647-L711]
    │
    │ format_error 做了什么：
    │ 1. 找到 "execute_custom_code()" 在 traceback 中的行号 → 初始截断点
    │ 2. 找到 block_uuid 或 "execute_block_function" 在 traceback 中的行号 → 结束截断点
    │ 3. 删除中间 Mage 内部堆栈帧，只保留用户代码相关的错误信息
    │ 4. 将 ANSI 深蓝色 [0;34m 替换为黄色 [0;33m（提高可读性）
    │
    ▼
前端收到格式化后的错误消息，显示在 Block 输出面板
```

#### 5.1.2 上游 Block 未执行

在 `__execute_block` 触发前，`Block.execute_sync` 检查上游状态（[block/__init__.py#L1499-L1518](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1499-L1518)）：

```python
not_executed_upstream_blocks = list(
    filter(lambda b: b.status == BlockStatus.NOT_EXECUTED, self.upstream_blocks)
)
if len(not_executed_upstream_blocks) > 0:
    raise Exception(
        f"Block {self.uuid}'s upstream blocks have not been executed yet. "
        f'Please run upstream blocks {upstream_block_uuids} before running the current block.'
    )
```

该异常由 `execute_custom_code.py` 捕获后通过 Jupyter kernel 的 error 通道回传。

#### 5.1.3 Pipeline 执行错误

在 `__execute_pipeline` 的子进程模式中（[websocket_server.py#L117-L135](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/websocket_server.py#L117-L135)）：

```python
try:
    pipeline.execute_sync(...)
    add_pipeline_message('...execution complete.', execution_state='idle')
except Exception:
    trace = traceback.format_exc().splitlines()
    add_pipeline_message('...execution failed with error:')
    add_pipeline_message(trace, execution_state='idle')  # ← idle 表示执行结束
```

#### 5.1.4 认证/权限错误

在 `on_message` 入口（[websocket_server.py#L213-L250](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/websocket_server.py#L213-L250)）：

```python
if not valid or DISABLE_NOTEBOOK_EDIT_ACCESS == 1:
    return self.send_message(dict(
        data=ApiError.UNAUTHORIZED_ACCESS['message'],
        execution_state='idle',
        type=DataType.TEXT_PLAIN,
    ))
```

### 5.2 Magic Kernel 路径的错误处理

#### 5.2.1 代码执行层（execution.py）

| 异常类型 | 状态 | 错误回传 |
|---------|------|---------|
| `StopAsyncIteration`（用户中断） | `CANCELLED` | `ErrorDetails.from_current_error(err)` |
| 其他 `Exception` | `ERROR` | `ErrorDetails.from_current_error(err)` 完整堆栈 |
| finally 块 | — | `main_queue.put(uuid)` + `queue.put(None)` 确保消费端不被阻塞 |

#### 5.2.2 进程池提交层（process.py）

```python
def execute_message(...):
    try:
        asyncio.run(execute_code_async(...))
    except Exception as err:
        print(f'[Process.execute_code:{uuid}] Error: {err}')  # 兜底日志
```

#### 5.2.3 ReaderThread 桥接层

```python
except Empty:
    pass                              # 空队列正常
except Exception as err:
    if is_debug():
        print(f'[ReaderThread] ERROR: {err}')  # 不 kill 线程
```

#### 5.2.4 Kernel 管理层

| 场景 | 处理 |
|-----|------|
| 终止超时（10s） | `force_terminate()` 暴力杀进程 |
| 终止后清理 Lock | `acquire(blocking=False)` 非阻塞尝试，失败则跳过 |
| 查询已死进程状态 | `psutil.NoSuchProcess` → 跳过 |
| drain 写队列 | 只清除当前 uuid 消息，其他 uuid 回推 |

### 5.3 execute_custom_code 内部的错误处理

[execute_custom_code.py](file:///d:/fz/0601/solo-dogfeeding/code/329-mage-ai/mage_ai/server/utils/execute_custom_code.py) 是运行在 Jupyter Kernel 内部的框架代码，其错误处理：

1. **`block.execute_with_callback(**options)`**：内部调用 `Block.execute_sync`，该方法对上游未执行、动态块计算等场景抛出异常
2. **动态子块执行**（`run_tasks`）：每个子块独立执行，通过 `send_status_update` 报告进度（如 "3 of 10 dynamic child blocks completed."），任何子块异常都会中断后续子块
3. **输出序列化**：`simplejson.dumps(..., default=encode_complex, ignore_nan=True)` 处理不可序列化类型

---

## 六、完整链路时序图

```
用户操作                  后端处理                          内核/存储
──────                  ────────                          ─────────

1. 创建 Block ─────────► Block.create()
                         │
                         ├─► load_template() ──────────► .py 文件落盘
                         │   fetch_template_source()
                         │   write_template()
                         │
                         └─► 返回 Block 元数据 ───────► 前端渲染编辑器

2. 编辑代码 ───────────► (前端本地编辑，未提交)

3. 点击运行 ───────────► WebSocket.on_message()
                         │
                         ├─► 获取 block.content / custom_code
                         ├─► add_execution_code() 包装
                         │   （嵌入 execute_custom_code.py）
                         ├─► add_internal_output_info() 追加
                         │   （嵌入 custom_output.py）
                         │
                         ├─► [Jupyter 路径]
                         │   client.execute(code) ──────► ipykernel 执行
                         │                                  │
                         │   get_messages() ◄────────────── iopub 回传
                         │   parse_output_message()
                         │   WebSocketServer.send_message()
                         │   client.write_message() ────► 前端显示输出
                         │
                         └─► [Magic 路径]
                             kernel.run(message) ────────► 进程池 exec
                                                            │
                             EventStreamHandler ◄────────── ReaderThread 桥接
                             SSE 推送 ──────────────────────► 前端显示输出

4. 出错时 ◄──────────── error 消息
                         │
                         ├─► [Jupyter] format_error()
                         │   过滤内部堆栈 + ANSI 着色
                         │
                         └─► [Magic] ErrorDetails
                             完整错误信息 + CANCELLED/ERROR 状态
```

---

## 七、关键设计洞察

### 7.1 代码的「二次模板化」

模板生成的代码（Templates 产物）在执行时又被嵌入到 **执行框架模板**（`execute_custom_code.py` + `custom_output.py`）中，形成「模板套模板」：

```
Templates 生成 ──► 用户代码字符串
                       │
                       ▼
add_execution_code() ──► execute_custom_code.py 模板（code=r'''{用户代码}'''）
                       │
                       ▼
add_internal_output_info() ──► 追加 custom_output.py（捕获最后表达式并序列化）
                       │
                       ▼
client.execute() ──► 完整可执行代码字符串
```

这个设计使得：
- 用户代码在 **Block 上下文**中执行（有 Pipeline/Block 对象、global_vars、日志器等）
- 输出自动被 **格式化与采样**（DataFrame 截断、序列化、类型推断）
- 执行状态自动被 **跟踪与更新**（Block 状态机、数据库记录）

### 7.2 两条路径的本质区别

| | Jupyter 路径 | Magic 路径 |
|---|---|---|
| **代码可见性** | 经过 `add_execution_code` + `add_internal_output_info` 重包装 | 原样传入 `kernel.run(message)` |
| **命名空间** | 单一持久命名空间（跨执行共享变量） | 每次执行独立 `exec_globals = {}` |
| **输出格式化** | Jupyter 消息协议 + `parse_output_message` + `format_error` | `ExecutionResult` 数据类 + SSE |
| **当前定位** | 主路径（Notebook 交互） | 实验性路径（独立并发执行） |

### 7.3 错误回传的层次差异

**Jupyter 路径**：
- 错误在 ipykernel 内部捕获 → 生成结构化 `error` 消息（含 traceback 列表）
- `format_error()` 专门过滤 Mage 内部堆栈帧（`execute_custom_code` 到 `block_uuid` 之间的行），只展示用户相关部分
- 这是因为 `execute_custom_code.py` 的包装增加了大量框架堆栈，不过滤会让用户困惑

**Magic 路径**：
- 错误在 `execute_code_async` 的 `except Exception` 中捕获
- 使用 `ErrorDetails.from_current_error(err)` 保留完整信息
- 因为 Magic 路径不做代码包装，堆栈本身就是用户代码的，无需过滤

### 7.4 状态回传的保序机制

**Jupyter**：通过 `msg_id` 关联，`running_executions_mapping[msg_id] = {block_uuid, block_type}`，确保多次执行的输出不会混淆。

**Magic**：通过 `uuid`（kernel ID）分区，`execution_result_queue[uuid]` 是每个 kernel 独立的 `FasterQueue`，`EventStreamHandler` 按 URL 中的 uuid 取对应队列。`None` 哨兵标记单次执行流的结束。

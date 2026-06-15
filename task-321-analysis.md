# Mage AI Notebook 风格运行 - 内部协作机制分析

## 概述

Mage AI 的 Notebook 风格运行对外表现为交互式的 Block 执行界面，用户可以逐个运行数据块并即时看到结果。内部通过 **多进程内核（Magic Kernel） + 变量持久化 + 事件流推送** 的三层架构实现，数据从上游 Block 流入，经内核进程执行，最终通过 SSE 事件流推送到前端。

---

## 一、数据怎么进来（入口层）

### 1.1 触发入口

用户在前端点击「运行 Block」按钮，触发执行请求。有两条执行路径：

| 路径 | 场景 | 核心入口 |
|------|------|----------|
| **Pipeline 调度** | 完整 Pipeline 运行 | `BlockExecutor.execute()` → `block.execute_block()` |
| **Notebook 交互** | 单个 Block 调试运行 | KernelManager → Kernel.run() → Process.start() |

### 1.2 Block 执行的入口方法

`Block.execute_block()` 是 Block 执行的统一入口，位于 [block/__init__.py#L1855-L1931](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1855-L1931)。

执行流程：
```
execute_block()
  └── __get_outputs_from_input_vars()  # 第一步：获取上游输入数据
  └── _execute_block()                 # 第二步：执行 Block 代码
  └── store_variables()                # 第三步：存储输出变量
```

### 1.3 上游数据获取

`__get_outputs_from_input_vars()` 负责从上游 Block 拉取数据：

- 调用 `fetch_input_variables()` 从变量存储中读取
- 支持从内存缓存（`block_run_outputs_cache`）或磁盘读取
- 数据被组装为 `input_vars` 列表和 `kwargs_vars` 字典
- 同时生成 `outputs_from_input_vars` 字典，包含 `df_1`, `df_2` 等别名，注入到 exec 全局作用域

### 1.4 Kernel 初始化（Magic Kernel）

Magic Kernel 是 Mage 自研的轻量级内核，替代传统的 Jupyter Kernel：

**核心组件** [kernels/magic/kernels/models.py#L20-L210](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/kernels/models.py#L20-L210)：

```
Kernel
  ├── pool: Pool(processes=N)           # 多进程池，实际执行代码
  ├── context_manager: SyncManager      # 跨进程共享状态管理器
  │     ├── read_queue: Queue           # 子进程 → 主线程 的结果队列
  │     ├── shared_dict: DictProxy      # 共享字典
  │     ├── shared_list: ListProxy      # 共享列表
  │     └── lock: Lock                  # 进程锁
  ├── reader_thread: ReaderThread       # 读取队列并转发
  ├── stop_event / stop_event_pool      # 停止信号
  └── write_queue: FasterQueue          # 内核 → 事件流 的输出队列
```

**KernelManager** 是单例管理器，维护所有 Kernel 实例 [kernels/magic/kernels/manager.py#L15-L106](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/kernels/manager.py#L15-L106)：
- `kernels = {}` 存储 uuid → Kernel 映射
- 支持 interrupt、terminate、restart 操作

---

## 二、数据怎么流转（执行层）

### 2.1 整体架构图

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  Frontend    │────▶│  EventStreamHandler  │────▶│  KernelManager   │
│  (SSE消费)   │     │  (SSE 事件流)        │     │  (内核管理)      │
└──────────────┘     └──────────────────────┘     └────────┬─────────┘
          ▲                                               │
          │                                               ▼
          │                                     ┌──────────────────┐
          │                                     │  Kernel          │
          │                                     │  ├── pool (进程池)│
          │                                     │  └── read_queue  │
          │                                     └────────┬─────────┘
          │                                              │
          │                                              ▼
          │                                     ┌──────────────────┐
          │                                     │  ReaderThread    │
          │                                     │  (队列转发)      │
          │                                     └────────┬─────────┘
          │                                              │
          └──────────────────────────────────────────────┘
                           write_queue
```

### 2.2 代码执行流程

**1. 提交执行任务** [process.py#L99-L129](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/process.py#L99-L129)

`Kernel.run(message)` 创建 `Process` 对象，通过 `pool.apply_async()` 提交到进程池：
- `message`: 要执行的代码字符串
- `read_queue`: 子进程写入结果的队列
- `stop_event_pool` + `self.stop_event`: 双停止事件（全局 + 单个任务）

**2. 子进程执行** [execution.py#L48-L159](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/execution.py#L48-L159)

`execute_code_async()` 在子进程中运行：

```python
# 编译代码
compiled_code = compile(message, '<string>', 'exec')

# 启动 stdout 读取线程（后台持续读取打印输出）
reader_thread = Thread(target=read_stdout_continuously, ...)

# 重定向 stdout 到 AsyncStdout
with redirect_stdout(async_stdout):
    exec(compiled_code, exec_globals)  # 执行完整代码块

# 单独计算最后一个表达式（类似 Notebook 的 Out 单元格）
last_expr = code_lines[-1].strip()
if last_expr and not last_expr.startswith('#'):
    compiled_expr = compile(last_expr, '<string>', 'eval')
    output = eval(compiled_expr, exec_globals)

# 发送成功结果 + 就绪状态
queue.put(ExecutionResult(status=SUCCESS, type=DATA, output=...))
queue.put(ExecutionResult(status=READY, type=STATUS, ...))
```

关键点：
- `exec_globals` 字典作为执行的全局命名空间
- 代码块完整执行后，额外 eval 最后一行表达式作为输出值
- stdout 通过 `AsyncStdout` 捕获，由独立线程持续读取并推送

**3. stdout 实时捕获** [execution.py#L18-L45](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/execution.py#L18-L45)

`read_stdout_continuously()` 在后台线程中运行：
- 每 50ms 检查一次 `AsyncStdout` 缓冲区
- 有内容就按行分割，封装为 `ExecutionResult(type=STDOUT)`
- 放入 read_queue，最终推送到前端

**4. 队列转发** [threads/reader.py#L11-L54](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/threads/reader.py#L11-L54)

`ReaderThread` 持续从 `read_queue` 读取结果，转发到 `write_queue`：
- read_queue 是 SyncManager 管理的跨进程队列
- write_queue 是主线程的高性能队列（FasterQueue）

### 2.3 Block 内部数据流转

`Block._execute_block()` 方法 [block/__init__.py#L1966-L2098](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py#L1966-L2098) 是实际执行代码的地方：

```
_execute_block()
  ├── 合并结果字典 results = merge_dict({...decorators...}, outputs_from_input_vars)
  ├── exec(self.content, results)          # 执行 Block 代码
  ├── 提取装饰器函数（@data_loader 等）
  ├── 执行 preprocessor 函数
  ├── 执行主函数（block_function）
  └── 返回 outputs 列表
```

**装饰器机制**：
- `@data_loader`, `@transformer`, `@data_exporter` 等装饰器标记函数
- 代码 `exec` 执行后，从 `results` 字典中提取被装饰的函数
- 调用 `execute_block_function()` 执行主函数，传入 `input_vars` 作为参数

### 2.4 变量存储与读取

**VariableManager** 管理变量的持久化 [variable_manager.py#L33-L150](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py#L33-L150)：

```
VariableManager
  ├── add_variable()      # 写入变量到磁盘
  ├── get_variable()      # 从磁盘读取变量
  └── storage: LocalStorage
```

**Block 间数据传递**：
1. 上游 Block 执行完成 → `store_variables()` 写入磁盘
2. 下游 Block 执行前 → `fetch_input_variables()` 从磁盘读取
3. 变量按 `pipeline_uuid/block_uuid/variable_uuid` 层级存储

**变量类型**：
- DataFrame / Series (pandas, polars)
- 基本类型（int, str, dict, list）
- 生成器（Generator） - 支持流式处理
- 可迭代对象 - 支持分批处理

---

## 三、数据怎么结束（出口层）

### 3.1 执行结果类型

`ExecutionResult` 封装了所有执行输出 [magic/models.py#L32-L56](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/models.py#L32-L56)：

| 字段 | 说明 |
|------|------|
| `type` | 结果类型：`STDOUT` / `DATA` / `STATUS` |
| `status` | 执行状态：`RUNNING` / `SUCCESS` / `ERROR` / `CANCELLED` / `READY` |
| `output` | 数据输出（最后一个表达式的值） |
| `error` | 错误信息（ErrorDetails） |
| `data_type` | 数据类型：`TEXT_PLAIN` 等 |
| `process` | 进程详情（pid, 状态等） |

### 3.2 结束信号

执行完成时，子进程按顺序发送：

1. **STDOUT 类型**：执行过程中持续产生的打印输出
2. **RUNNING 状态**：代码块执行完成（在 redirect_stdout 块内）
3. **DATA 类型 + SUCCESS 状态**：携带最后表达式的输出值
4. **STATUS 类型 + READY 状态**：标记内核就绪，可以接受下一个任务
5. **None 哨兵值**：放入队列，标记本次执行彻底结束

```python
queue.put(ExecutionResult(status=RUNNING, type=STATUS))     # 第2步
queue.put(ExecutionResult(status=SUCCESS, type=DATA, output=last_output))  # 第3步
queue.put(ExecutionResult(status=READY, type=STATUS))       # 第4步
queue.put(None)  # 哨兵值                                    # 第5步
```

### 3.3 事件流推送（SSE）

`EventStreamHandler` 通过 Server-Sent Events 向前端推送结果 [events/stream.py#L20-L63](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py#L20-L63)：

- 前端建立 SSE 连接，指定 kernel uuid
- 后端循环从 `write_queue` 读取结果
- 封装为 `EventStream` 对象，JSON 序列化后推送
- 每 100ms 轮询一次队列

### 3.4 错误处理

在 `execute_code_async()` 的 try-except 中捕获：

| 异常类型 | 处理方式 | 状态 |
|----------|----------|------|
| `StopAsyncIteration` | 用户取消 | `CANCELLED` |
| 其他 Exception | 捕获并记录错误栈 | `ERROR` |

错误详情通过 `ErrorDetails.from_current_error(err)` 封装，包含：
- 错误消息
- 堆栈跟踪
- 错误类型

### 3.5 变量持久化

Block 执行成功后，`execute_block_function()` 内部调用 `store_variables()` 将输出持久化：

- 支持同步和异步两种方式
- 动态处理 DataFrame 分批存储
- 支持 Spark DataFrame 转换
- 写入磁盘后，下游 Block 可通过变量名读取

---

## 四、核心协作机制总结

### 4.1 三层队列架构

| 层级 | 队列 | 作用 | 生产者 | 消费者 |
|------|------|------|--------|--------|
| L1 | `read_queue` | 子进程 → 内核主线程 | 子进程执行代码 | ReaderThread |
| L2 | `write_queue` | 内核 → 事件流 | ReaderThread | EventStreamHandler |
| L3 | SSE 连接 | 后端 → 前端 | EventStreamHandler | 前端 UI |

### 4.2 双线程 + 多进程模型

```
主线程 (Main Thread)
  ├── KernelManager (单例)
  ├── Kernel (多个，按 uuid 管理)
  │     ├── ReaderThread (每个 kernel 一个，读队列)
  │     └── Pool (N 个工作进程)
  │           └── Process (子进程，实际执行代码)
  │                 └── reader_thread (子进程内线程，读 stdout)
  └── EventStreamHandler (Tornado 异步 handler)
```

### 4.3 状态流转

```
INIT → RUNNING → SUCCESS → READY
              → ERROR   → READY
              → CANCELLED
```

- **RUNNING**: 代码正在执行，stdout 持续输出
- **SUCCESS/ERROR**: 执行完成，携带结果或错误
- **READY**: 内核就绪，可接受新任务
- **CANCELLED**: 用户主动中断

---

## 五、关键代码文件索引

| 文件 | 核心职责 |
|------|----------|
| [kernels/magic/kernels/manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/kernels/manager.py) | Kernel 管理器（单例） |
| [kernels/magic/kernels/models.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/kernels/models.py) | Kernel 核心实现 |
| [kernels/magic/process.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/process.py) | 执行进程封装 |
| [kernels/magic/execution.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/execution.py) | 代码执行逻辑 |
| [kernels/magic/models.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/models.py) | ExecutionResult 等数据模型 |
| [kernels/magic/threads/reader.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/threads/reader.py) | 队列读取转发线程 |
| [kernels/magic/queues/manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/queues/manager.py) | 全局结果队列管理 |
| [kernels/magic/stdout.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/kernels/magic/stdout.py) | 异步 stdout 捕获 |
| [data_preparation/models/block/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/models/block/__init__.py) | Block 执行核心逻辑 |
| [data_preparation/executors/block_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/executors/block_executor.py) | Block 执行器（调度层） |
| [data_preparation/variable_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/data_preparation/variable_manager.py) | 变量持久化管理 |
| [server/events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/321-mage-ai/mage_ai/server/events/stream.py) | SSE 事件流处理器 |

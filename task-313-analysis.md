# Streaming Pipeline 代码深度分析

## 一、整体架构概览

Mage AI 的 Streaming Pipeline 采用了**分层架构 + 工厂模式 + 模板方法模式**的组合设计。整个系统可以划分为以下 5 个核心层级：

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 5: 入口调度层 (Pipeline Model)                        │
│  Pipeline.execute() → 根据 type 路由到对应 Executor          │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: 执行器层 (Pipeline Executor)                       │
│  StreamingPipelineExecutor → 控制整体生命周期、重试、日志     │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: 工厂实例化层 (Factory)                             │
│  SourceFactory / SinkFactory → 根据配置创建具体实例          │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: 抽象基类层 (Base Class)                            │
│  BaseSource / BaseSink → 定义统一接口、通用能力（检查点、缓冲）│
├─────────────────────────────────────────────────────────────┤
│  Layer 1: 具体连接器层 (Connector)                           │
│  KafkaSource / PostgresSink / ... → 对接具体中间件            │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、职责分层详解

### 2.1 Layer 5: 入口调度层 — Pipeline Model

**核心文件**: [pipeline.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/models/pipeline.py)

**核心代码定位**: [pipeline.py#L798-L813](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/models/pipeline.py#L798-L813)

**职责**:
- 作为外部调用的统一入口，所有 Pipeline（Batch/Integration/Streaming）都通过 `Pipeline.execute()` 触发
- 根据 `self.type` 字段进行类型分发，当 `type == PipelineType.STREAMING` 时，动态导入并实例化 `StreamingPipelineExecutor`
- 提供执行上下文（如 `pipeline_variables_dir`，即检查点和缓冲区的存储路径）

**关键设计点**:
- Streaming Pipeline 的 `executor_count` 属性返回自定义值（而非默认 1），表明其支持多执行器并行部署
- 入口处采用**延迟导入**（在方法内部 import），避免循环依赖

---

### 2.2 Layer 4: 执行器层 — StreamingPipelineExecutor

**核心文件**: [streaming_pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py)

**核心代码定位**: [streaming_pipeline_executor.py#L28-L319](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L28-L319)

**继承关系**: `StreamingPipelineExecutor → PipelineExecutor`

#### 2.2.1 核心职责拆解

| 方法 | 职责 |
|------|------|
| `__init__()` + `parse_and_validate_blocks()` | **Block 拓扑结构校验**，确保 DAG 是 Source→Transformer*→Sink 的合法树形结构 |
| `execute()` | **总控入口**：初始化日志、加载重试配置、包裹重试装饰器、捕获顶级异常 |
| `__execute_in_python()` | **核心运行时**：初始化 Source/Sink 实例、构建 DFS 处理链、启动长轮询消费循环 |
| `__update_pipeline_run_status()` | **状态持久化**：更新数据库中 PipelineRun 的状态，发送失败通知 |

#### 2.2.2 Block 拓扑校验规则 (`parse_and_validate_blocks`)

Streaming Pipeline 对 Block 拓扑有严格约束，不允许任意 DAG：

```
合法结构示例:
Source (DataLoader)
   ├── Transformer A ──→ Sink X (DataExporter)
   ├── Transformer B ──→ Sink Y (DataExporter)
   └───────────────────→ Sink Z (DataExporter)
```

**校验规则**:
1. **Source 约束**: 必须是 `DATA_LOADER` 类型，且只能有 **1 个**，必须是根节点（无上游），至少有 1 个下游
2. **Sink 约束**: 必须是 `DATA_EXPORTER` 类型，必须是叶子节点（无下游），必须有且仅有 1 个上游
3. **Transformer 约束**: 必须是 `TRANSFORMER` 类型，必须有且仅有 1 个上游（支持 0~N 个下游）

代码位置: [streaming_pipeline_executor.py#L35-L73](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L35-L73)

#### 2.2.3 与基类 PipelineExecutor 的差异

| 维度 | PipelineExecutor（Batch） | StreamingPipelineExecutor |
|------|--------------------------|---------------------------|
| 执行模式 | 异步并发调度独立 BlockRun | 长驻进程 + 消息驱动回调 |
| 调度单元 | Block 级别（每个 Block 独立执行） | Pipeline 级别（整个 DAG 串成一条处理链） |
| 完成条件 | 所有 BlockRun 完成 | 永不停止（直到进程终止或异常） |
| 状态管理 | 每个 BlockRun 独立状态 | 仅 PipelineRun 级别状态 |
| 重试粒度 | Block 级别 | 整个 Pipeline 级别 |

基类代码: [pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/pipeline_executor.py)

---

### 2.3 Layer 3: 工厂实例化层 — SourceFactory / SinkFactory

**核心文件**: 
- [source_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/source_factory.py)
- [sink_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/sink_factory.py)

#### 2.3.1 双模式实例化

工厂提供两种实例化方式，对应 Block 的两种语言类型：

**模式 1: YAML 配置模式**（BlockLanguage 非 Python）
```python
# SourceFactory.get_source(config, checkpoint_path=...)
# 根据 config['connector_type'] 分发到具体 Source 类
```

**模式 2: Python 代码模式**（BlockLanguage == Python）
```python
# SourceFactory.get_python_source(content, global_vars=...)
# 通过 exec() 执行用户代码，收集 @streaming_source 装饰的类
```

Python 模式的核心机制 —— **装饰器收集模式**:

代码位置: [decorators.py#L108-L125](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/decorators.py#L108-L125)

```python
# 用户代码:
@streaming_source
class MyKafkaSource(BasePythonSource):
    ...

# 工厂内部:
decorated_sources = []
exec(content, {'streaming_source': collect_decorated_objs(decorated_sources)})
# decorated_sources[0] 即为被装饰的类
```

#### 2.3.2 支持的连接器类型

定义于: [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/constants.py)

| 类型 | Source | Sink |
|------|--------|------|
| Kafka | ✅ | ✅ |
| RabbitMQ | ✅ | ✅ |
| ActiveMQ | ✅ | ✅ |
| MongoDB | ✅ | ✅ |
| Kinesis | ✅ | ✅ |
| NATS JetStream | ✅ | - |
| InfluxDB | ✅ | ✅ |
| Amazon SQS | ✅ | - |
| Azure Event Hub | ✅ | - |
| Google Cloud Pub/Sub | ✅ | ✅ |
| PostgreSQL | - | ✅ |
| Elasticsearch | - | ✅ |
| OpenSearch | - | ✅ |
| OracleDB | - | ✅ |
| Amazon S3 | - | ✅ |
| Google Cloud Storage | - | ✅ |
| Azure Data Lake | - | ✅ |
| Generic IO (BigQuery/Snowflake/...) | - | ✅ (通过 GenericIOSink) |

---

### 2.4 Layer 2: 抽象基类层 — BaseSource / BaseSink

#### 2.4.1 BaseSource

**核心文件**: [base.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/base.py)

**模板方法模式** —— 基类定义初始化骨架，子类实现具体客户端逻辑：

```
__init__(config)
    ├── config_class.load(config)         # 配置加载（如果有 config_class）
    ├── read_checkpoint()                 # 读取检查点（从本地 JSON 文件）
    ├── init_client()                     # ← 子类实现：初始化客户端连接
    └── test_connection()                 # 测试连通性（非测试环境下）
```

**三种消费模式** (SourceConsumeMethod):

| 模式 | 对应方法 | 适用场景 |
|------|---------|---------|
| `BATCH_READ` | `batch_read(handler)` | 批量拉取后一次性处理（如 Kafka poll） |
| `READ` | `read(handler)` | 逐条同步消费（如 for msg in consumer） |
| `READ_ASYNC` | `read_async(handler)` | 异步逐条消费（async for 模式） |

基类默认提供 `read_async` 的 fallback 实现（同步转异步），子类可按需覆盖。

**检查点机制**:
- 检查点存储路径: `<variables_dir>/pipelines/<pipeline_uuid>/streaming_checkpoint`
- 格式: JSON 文件，内容由子类自定义（如 Kafka 的 offset、Kinesis 的 sequence number）
- 基类提供 `read_checkpoint()` / `update_checkpoint()` 通用读写方法

#### 2.4.2 BaseSink

**核心文件**: [base.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/base.py)

**缓冲区机制** (Buffer) —— 为了应对下游暂时不可用的情况：

```
write(message)
    └── batch_write([message])           # 单条写入降级为批量写入
        ├── [子类实现] 尝试写入下游
        └── 如果失败: 写入本地 buffer 文件
```

缓冲区存储路径: `<variables_dir>/pipelines/<pipeline_uuid>/buffer`

**缓冲区关键特性**:
- 持久化: 每行一条 JSON，append-only 写入
- 启动恢复: `__init__` 时自动读取已有 buffer
- 超时判断: `has_buffer_timed_out(buffer_timeout_seconds)` 用于判断是否该强制 flush
- 清空: `clear_buffer()` 在成功写入后调用

**消息格式 V2** (`_is_message_format_v2`):
- 两种消息格式兼容:
  - V1: 直接是数据本身 `{...}`
  - V2: `{"data": {...}, "metadata": {...}}` 包装结构
- `metadata` 可携带 timestamp、index、key 等额外信息

#### 2.4.3 Python 用户自定义基类

- [base_python.py (Source)](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/base_python.py)
- [base_python.py (Sink)](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/base_python.py)

这两个类是 `BaseSource`/`BaseSink` 的"空配置"子类，供用户通过 `@streaming_source`/`@streaming_sink` 装饰器编写自定义代码时继承。它们跳过了 `config_class` 的配置加载流程。

---

### 2.5 Layer 1: 具体连接器层（以 KafkaSource + PostgresSink 为例）

#### 2.5.1 KafkaSource 典型实现

**核心文件**: [kafka.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/kafka.py)

**配置类分层**:
```
KafkaConfig
    ├── bootstrap_server, consumer_group, topic(s)
    ├── SSLConfig (cafile, certfile, keyfile, ...)
    ├── SASLConfig (PLAIN/SCRAM/OAUTHBEARER 三种机制)
    └── SerDeConfig (JSON/Protobuf/Avro/RAW_VALUE 四种序列化方式)
```

**关键流程**:
1. `init_client()`: 根据安全协议构建 `KafkaConsumer`，处理 SSL/SASL/OAuth 认证，加载 Protobuf Schema 或 Avro Schema Registry
2. `_handle_offsets()`: 支持根据 `offset` 配置精确重置消费位置（支持 int/timestamp/beginning/end 四种模式）
3. `batch_read()`: 循环调用 `consumer.poll(max_records=batch_size, timeout_ms=...)` → 反序列化 → handler 回调 → `consumer.commit()`
4. `_convert_message()`: 根据 `include_metadata` 决定是否包装为 V2 格式

#### 2.5.2 PostgresSink 典型实现

**核心文件**: [postgres.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/postgres.py)

**关键流程**:
1. `init_client()`: 通过 Mage 内置的 `Postgres` IO 客户端建立连接
2. `batch_write(messages)`: 将消息列表转为 DataFrame → 调用 `postgres_client.export()` 以 APPEND 模式写入 → 支持唯一约束冲突策略（`unique_conflict_method`）
3. `destroy()`: 安全关闭连接

---

## 三、关键数据流分析

### 3.1 端到端数据流转全景图

```
┌─────────────┐     messages      ┌──────────────────────┐
│   Source    │ ────────────────→ │ outputs_by_block     │
│ (DataLoader)│                   │ {source_uuid: msgs}  │
└─────────────┘                   └──────────┬───────────┘
                                              │
                                              ▼
                                   ┌──────────────────────┐
                                   │ handle_batch_events_ │
                                   │ recursively()        │
                                   │ DFS 遍历下游 Blocks  │
                                   └──────────┬───────────┘
                                              │
                          ┌───────────────────┼───────────────────┐
                          │                   │                   │
                          ▼                   ▼                   ▼
                ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐
                │ Transformer A   │  │ Transformer B   │  │ Direct Sink  │
                │ execute_block() │  │ execute_block() │  │ batch_write()│
                └────────┬────────┘  └────────┬────────┘  └──────────────┘
                         │                    │
                         ▼                    ▼
                ┌─────────────────┐  ┌─────────────────┐
                │  Sink X         │  │  Sink Y         │
                │  batch_write()  │  │  batch_write()  │
                └─────────────────┘  └─────────────────┘
```

### 3.2 数据处理链核心代码解析

代码位置: [streaming_pipeline_executor.py#L189-L258](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L189-L258)

#### 3.2.1 深度拷贝（`__deepcopy`）

Streaming 模式下，同一份上游数据可能被多个下游 Transformer/Sink 消费。为了防止某个节点修改数据影响其他节点，系统在传递给每个下游之前都会对数据进行 **深拷贝**：

```python
# 深拷贝降级策略:
# 1. None → 直接返回
# 2. list → 递归逐个拷贝
# 3. 其他 → 尝试 copy.deepcopy，失败则降级为 copy.copy
```

#### 3.2.2 DFS 递归处理（`handle_batch_events_recursively`）

这是整个数据流的核心调度逻辑：

```python
def handle_batch_events_recursively(curr_block, outputs_by_block, **kwargs):
    curr_block_output = outputs_by_block[curr_block.uuid]
    
    for downstream_block in curr_block.downstream_blocks:
        if downstream_block.type == TRANSFORMER:
            # 执行 Transformer Block，结果存入 outputs_by_block
            outputs_by_block[downstream_block.uuid] = downstream_block.execute_block(
                input_args=[__deepcopy(curr_block_output)],
                ...
            )['output']
            
        elif downstream_block.type == DATA_EXPORTER:
            # 直接调用 Sink 的 batch_write
            sinks_by_uuid[downstream_block.uuid].batch_write(
                __deepcopy(curr_block_output)
            )
        
        # 递归处理下游的下游（Transformer 可以继续分支）
        if downstream_block.downstream_blocks:
            handle_batch_events_recursively(downstream_block, outputs_by_block, **kwargs)
```

**关键洞察**:
- Transformer 执行使用的是 Block 的通用 `execute_block()` 方法（与 Batch Pipeline 共享），因此可以复用所有 Block 执行逻辑（日志、变量、测试等）
- Sink 不通过 Block 执行，而是直接调用 `sink.batch_write()`，因为 Sink 的生命周期是长驻的（不是执行一次就结束）

#### 3.2.3 三种消费模式的入口差异

| 消费模式 | 入口 handler | 数据包装方式 |
|---------|-------------|-------------|
| BATCH_READ | `handle_batch_events(messages)` | `{source_uuid: messages}` (message 列表) |
| READ | `handle_event(message)` | `{source_uuid: [message]}` (单条包成列表) |
| READ_ASYNC | `handle_event_async(message)` | 同上，但是 async 函数 |

三种模式最终都汇聚到同一个 `handle_batch_events_recursively()`，保证了处理逻辑的一致性。

### 3.3 YAML 变量插值

对于非 Python 模式的 Block，Source/Sink 的配置支持 Jinja2 模板变量：

代码位置: [streaming_pipeline_executor.py#L312-L319](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L312-L319)

```python
config_file = Template(content).render(
    variables=lambda x: global_vars.get(x) if global_vars else None,
    **get_template_vars()
)
```

这允许在 YAML 配置中通过 `{{ variables('env_var') }}` 引用运行时变量。

---

## 四、异常分支与容错机制分析

### 4.1 异常处理层级总览

```
Level 0: Source/Sink 内部异常         → 缓冲 + 重试 (BaseSink buffer)
Level 1: 单条/单批消息处理异常         → 冒泡到消费循环
Level 2: 消费循环整体异常             → PipelineExecutor 重试装饰器
Level 3: 超出重试次数的致命异常        → 状态标记 FAILED + 通知
```

### 4.2 Level 3: Pipeline 级重试机制

代码位置: [streaming_pipeline_executor.py#L96-L136](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L96-L136)

**重试配置模型** ([retry.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/shared/retry.py)):
```python
@dataclass
class RetryConfig:
    retries: int = 0                    # 重试次数
    delay: int = 5                      # 初始延迟（秒）
    max_delay: int = 60                 # 最大延迟（秒）
    exponential_backoff: bool = True    # 是否指数退避
```

**重试决策逻辑**:
```python
if retry_config is None:
    retry_config = self.pipeline.retry_config or dict()
infinite_retries = False if retry_config else True
```

- **如果 Pipeline 配置了 retry_config**: 使用配置的有限次数重试，超出后标记 `PipelineRun.FAILED`
- **如果没有任何 retry_config**: `infinite_retries = True`，意味着**永不标记失败**，异常会不断重试（实际上是无限循环重启消费）

**重试装饰器实际行为** ([shared/retry.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/shared/retry.py)):

```python
while attempt <= max_attempts:  # max_attempts = retries + 1
    try:
        return func()
    except Exception:
        attempt += 1
        if attempt > max_attempts or total_delay + curr_delay >= max_delay:
            raise e  # 重试耗尽，抛出异常
        time.sleep(curr_delay)
        if exponential_backoff:
            curr_delay *= 2
```

### 4.3 失败通知与状态持久化

代码位置: [streaming_pipeline_executor.py#L277-L304](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L277-L304)

当重试耗尽且 `infinite_retries == False` 时：

1. 更新 `PipelineRun.status = FAILED`
2. 设置 `completed_at = now`
3. 上报使用统计 `UsageStatisticLogger().pipeline_run_ended()`
4. 提取异常信息和堆栈
5. 通过 `notification_sender.send_pipeline_run_failure_message()` 发送通知

### 4.4 资源清理保证

代码位置: [streaming_pipeline_executor.py#L261-L275](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L261-L275)

```python
try:
    # 长驻消费循环
    source.batch_read(...) / source.read(...) / source.read_async(...)
finally:
    source.destroy()          # 关闭 Source 连接
    for sink in sinks_by_uuid.values():
        sink.destroy()        # 关闭所有 Sink 连接
```

使用 `try/finally` 确保即使消费过程中抛出异常，Source 和 Sink 的连接也能被正确释放。此外 `BaseSource` 和 `BaseSink` 都实现了 `__del__` 析构函数，作为兜底调用 `destroy()`。

### 4.5 Sink 本地缓冲区容错

代码位置: [base.py#L54-L101](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/base.py#L54-L101)

BaseSink 提供了一套完整的本地缓冲机制，但**注意：它是可选的，需要具体 Sink 子类主动调用** `write_buffer()` 和 `read_buffer()`。

典型流程（子类需要自行实现）：
```
1. batch_write() 尝试写入下游
2. 如果写入失败 → self.write_buffer(failed_messages)
3. 下次 batch_write() 时 → 先尝试 flush 缓冲区内的数据
4. 启动时 → self.buffer = self.read_buffer() 恢复上次未写入的数据
```

### 4.6 异常分支汇总表

| 异常场景 | 触发位置 | 处理方式 | 后续行为 |
|---------|---------|---------|---------|
| Block 拓扑不合法 | `parse_and_validate_blocks()` | 直接 raise Exception | Pipeline 启动失败 |
| Factory 不支持的 connector_type | `SourceFactory.get_source()` / `SinkFactory.get_sink()` | raise Exception | Pipeline 启动失败 |
| Python 模式找不到装饰类 | `get_python_source()` / `get_python_sink()` | raise Exception | Pipeline 启动失败 |
| Source/Sink 客户端初始化失败 | `init_client()` | raise（Sink 会先调用 destroy） | Pipeline 启动失败，进入重试 |
| 单条消息处理异常 | Transformer 执行 / Sink 写入 | 向上冒泡 | 整个消费循环中断，进入 Pipeline 重试 |
| 消费循环中断 | 上述任何异常冒泡到 `__execute_in_python()` | finally 中 destroy 所有连接 | 由 @retry 装饰器控制重试 |
| 重试耗尽 | @retry 装饰器 | `__update_pipeline_run_status(FAILED)` | Pipeline 标记失败，发送通知 |

---

## 五、关键设计模式与实现细节

### 5.1 使用的设计模式

| 模式 | 应用位置 | 作用 |
|------|---------|------|
| 工厂方法 | SourceFactory / SinkFactory | 根据 connector_type 创建对应实例 |
| 模板方法 | BaseSource / BaseSink | 定义初始化/销毁骨架，子类实现具体客户端逻辑 |
| 策略模式 | SourceConsumeMethod (BATCH_READ/READ/READ_ASYNC) | 三种消费策略可互换 |
| 装饰器模式 | @retry, @streaming_source, @streaming_sink | 横切关注点（重试）与用户代码收集 |
| 组合模式 | Block DAG + DFS 递归遍历 | 树形拓扑的统一处理 |
| 适配器模式 | BasePythonSource / BasePythonSink | 适配无配置的用户自定义类 |

### 5.2 日志重定向机制

代码位置: [stream.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/shared/stream.py)

`StreamToLogger` 是一个伪装成 file-like 对象的类，将 stdout/stderr 的 `write()` 操作重定向到 Python Logger。配合 `redirect_stdout` / `redirect_stderr` 上下文管理器，实现了用户代码 print 输出的统一日志收集。

### 5.3 尚待实现的功能

从代码中的 TODO 可以看到 Streaming Pipeline 的规划方向：

1. **多 Source / 多 Sink 支持**: 当前代码限制只能有 1 个 Source，计划扩展
2. **Flink Pipeline 支持**: `__execute_in_flink()` 方法为空实现，未来可能接入 Flink 引擎
3. **自定义日志目的地**: `__init__` 中标注 TODO，当前仅支持默认日志输出

---

## 六、文件索引速查表

| 层级 | 文件 | 核心作用 |
|------|------|---------|
| 入口 | [pipeline.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/models/pipeline.py) | Pipeline 模型，Streaming 类型分发入口 |
| 执行器 | [streaming_pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py) | Streaming Pipeline 核心执行器 |
| 执行器基类 | [pipeline_executor.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/pipeline_executor.py) | Batch Pipeline 执行器，Streaming 继承自它 |
| 工厂 | [source_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/source_factory.py) | Source 实例化工厂 |
| 工厂 | [sink_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/sink_factory.py) | Sink 实例化工厂 |
| 基类 | [sources/base.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/base.py) | Source 抽象基类，检查点、消费模式 |
| 基类 | [sinks/base.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/base.py) | Sink 抽象基类，缓冲区、消息格式 |
| Python 基类 | [sources/base_python.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/base_python.py) | 用户自定义 Source 基类 |
| Python 基类 | [sinks/base_python.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/base_python.py) | 用户自定义 Sink 基类 |
| 装饰器 | [decorators.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/decorators.py) | @streaming_source / @streaming_sink 定义 |
| 常量 | [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/constants.py) | SourceType / SinkType 枚举 |
| 工具 | [stream.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/shared/stream.py) | StreamToLogger 日志重定向 |
| 重试 | [shared/retry.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/shared/retry.py) | @retry 装饰器实现 |
| 重试配置 | [shared/retry.py (data_preparation)](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/shared/retry.py) | RetryConfig 数据类 |
| Kafka Source | [kafka.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sources/kafka.py) | Kafka 消费端完整实现 |
| Postgres Sink | [postgres.py](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/streaming/sinks/postgres.py) | Postgres 写入端完整实现 |

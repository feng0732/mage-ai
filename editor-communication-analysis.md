# Mage AI 前端 Editor 通信深度分析（第三版：实时回传边界全覆盖）

> 本文档在前两版基础上，针对实时回传的**边界场景**进行穷举式核实：
> 1. 消息推送入口的所有来源——逐一识别 `send_message` 的 6 条调用路径
> 2. 错误栈路径中脱敏与 format_error 覆盖 data 的处理顺序
> 3. 运行保护（`disable_pipeline_edit_access`）下"保存但不一定发送执行消息"的分支逻辑

---

## 一、send_message 的所有调用来源（6 条路径）

`WebSocketServer.send_message()` 是后端推送消息到前端的唯一出口。经代码核实，共有 **6 条独立调用路径**：

### 全局调用路径图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        WebSocketServer.send_message()                           │
│                     websocket_server.py#L312-L399                                │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────────────────────┤
│  路径 1   │  路径 2   │  路径 3   │  路径 4   │  路径 5   │       路径 6          │
│ 未授权    │ 直接转发  │ 空白执行  │ 订阅器    │ 管道执行  │  取消/状态检查         │
│ 返回      │ output   │ 结果     │ 回调      │ 消息     │  publish_pipeline_msg  │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────────────────────┘
```

---

### 路径 1：未授权返回

**代码位置**：[websocket_server.py#L241-L250](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L241-L250)

```python
if not valid or DISABLE_NOTEBOOK_EDIT_ACCESS == 1:
    return self.send_message(
        dict(
            data=ApiError.UNAUTHORIZED_ACCESS['message'],
            execution_metadata=dict(block_uuid=message.get('uuid')),
            execution_state='idle',
            msg_id=str(uuid.uuid4()),
            type=DataType.TEXT_PLAIN,
        ),
    )
```

**消息特征**：

| 字段 | 值 | 说明 |
|------|-----|------|
| `data` | `ApiError.UNAUTHORIZED_ACCESS['message']` | 字符串，非列表 |
| `execution_metadata` | `{block_uuid: message.uuid}` | ★ 直接在 execution_metadata 中提供了 block_uuid |
| `execution_state` | `'idle'` | 立即标记为空闲 |
| `msg_id` | `str(uuid.uuid4())` | 随机生成 |
| `type` | `TEXT_PLAIN` | |

**三层处理后的命运**：

1. **消息过滤**：msg_id 有值、data 有值 → **通过**
2. **环境值脱敏**：data 是字符串，`filter_out_sensitive_data` 会先转为 `[data]` 再逐元素脱敏 → **会执行脱敏**
3. **执行元数据补充**：
   - `execution_metadata` 不为 None → **直接用作 msg_id_value**
   - 从中提取 `block_uuid = message.uuid`
   - 最终合并 `uuid=block_uuid` 到 output_dict → **前端可以按 uuid 分桶**
4. **推送判定**：`block_uuid` 存在 → **会推送 + 打日志**

> **关键**：未授权消息虽然不经过 `running_executions_mapping`（因为没有执行内核），但通过 `execution_metadata` 手动注入了 block_uuid，使得前端能正确地将 "Unauthorized" 消息关联到对应 block。

---

### 路径 2：直接转发（output 字段）

**代码位置**：[websocket_server.py#L252-L255](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L252-L255)

```python
output = message.get('output')
if output:
    self.send_message(output)
    return
```

**触发条件**：前端发送的 WebSocket 消息中包含 `output` 字段。

**消息特征**：

| 字段 | 值 | 说明 |
|------|-----|------|
| 整个 output | 前端自行构造的 dict | 不经过任何内核处理，直接回传 |

**三层处理后的命运**：

1. **消息过滤**：取决于 output 字典是否包含 `msg_id`、`data`/`error`/`execution_state`/`type`
   - 如果 output 缺少 `msg_id` → **直接丢弃**（L344-L346）
   - 如果 output 的四个核心字段全空 → **直接丢弃**
2. **环境值脱敏**：如果 `output.data` 存在 → **会执行脱敏**
3. **执行元数据补充**：
   - 无 `execution_metadata` → 走 `running_executions_mapping.get(msg_id, {})`
   - 如果 msg_id 不在映射表中 → `msg_id_value = {}` → `block_uuid=None, pipeline_uuid=None`
   - → **L392 判定 `if block_uuid or pipeline_uuid` 为 False → 不会推送到前端！**

> **⚠️ 边界发现**：如果前端通过 `output` 字段直接转发消息，且该 msg_id 不在 `running_executions_mapping` 中，也没有 `execution_metadata`，那么消息虽然通过了过滤和脱敏，但会在最终推送判定处被静默丢弃。这是一个**设计保护**而非 bug——只有与某个 block/pipeline 关联的 output 才有意义。

---

### 路径 3：空白执行结果（Scratchpad 无代码）

**代码位置**：[websocket_server.py#L449-L458](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L449-L458)

```python
if not custom_code and BlockType.SCRATCHPAD == block_type:
    self.send_message(
        dict(
            data='',
            execution_metadata=value,   # value = {block_type, block_uuid, replicated_block}
            execution_state='idle',
            msg_id=str(uuid.uuid4()),
            type=DataType.TEXT_PLAIN,
        ),
    )
```

**触发条件**：用户对 scratchpad block 按了运行，但编辑器中代码为空。

**消息特征**：

| 字段 | 值 | 说明 |
|------|-----|------|
| `data` | `''` (空字符串) | ← 伪造的 data，不是 None |
| `execution_metadata` | `{block_type, block_uuid, replicated_block}` | ★ 直接提供 |
| `execution_state` | `'idle'` | 立即标记空闲（未实际执行） |
| `msg_id` | 随机 UUID | |

**三层处理后的命运**：

1. **消息过滤**：msg_id 有值，data 是 `''`（不是 None）→ data 非空为 True → **通过**
   - 但 `should_filter_message` 的条件是 `data is None`，空字符串不算 None
2. **环境值脱敏**：`message.get('data')` = `''` → **空字符串是 falsy** → `if not message.get('data')` 为 True → **跳过脱敏**（正确行为：空字符串无需脱敏）
3. **执行元数据补充**：
   - `execution_metadata` 存在 → 用它作为 msg_id_value
   - 提取 `block_uuid`, `block_type` → 合并到 output_dict
4. **推送判定**：block_uuid 存在 → **会推送**

> **设计意图**：空代码的 scratchpad 也需要返回一个 `idle` 状态，让前端将 block 从 `runningBlocks` 中移除，避免 UI 一直显示"运行中"状态。

---

### 路径 4：订阅器回调（内核输出 → 主路径）

**代码位置**：[server.py#L745-L749](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/server.py#L745-L749)

```python
get_messages(
    lambda content: WebSocketServer.send_message(
        parse_output_message(content),
    ),
)
```

这是**最高频**的调用路径。每次 Jupyter 内核产生 IOPub 消息，subscriber 在独立线程中轮询到后，通过回调触发。

**消息特征**（来自 `parse_output_message` 输出）：

| 字段 | 值 | 说明 |
|------|-----|------|
| `msg_id` | `parent_header.msg_id` | 关联到 `client.execute()` 的返回值 |
| `data` | 规范化后：list/str/None | 取决于 msg_type |
| `error` | traceback 列表 或 None | error 类型时有值 |
| `execution_state` | `'busy'`/`'idle'`/None | status 类型时有值 |
| `type` | DataType 枚举 | TEXT_PLAIN / IMAGE_PNG / TEXT_HTML / TEXT 等 |
| `metadata` | 原始 metadata | 透传 |

**三层处理后的命运**：

1. **消息过滤**：
   - `msg_id` 可能缺失 → 丢弃
   - 四要素全空 → 丢弃
   - FloatProgress display_data → 丢弃
2. **环境值脱敏**：data 中每行文本都会经过 `filter_out_env_var_values`
3. **执行元数据补充**：
   - 无 `execution_metadata` → 查 `running_executions_mapping[msg_id]`
   - 如果该 msg_id 在映射表中 → 提取 block_type/block_uuid/pipeline_uuid → 合并
   - **如果不在映射表中** → `msg_id_value = {}` → 推送判定为 False → 静默丢弃

> **边界说明**：内核的 status 消息（busy/idle）的 msg_id 经常无法映射到 running_executions_mapping，这类消息会被静默丢弃。前端的 execution_state 变更是通过 stream 类型的消息间接获得的（当 parse_output_message 解析出 execution_state 时），而非直接从 Jupyter status 消息获得。

---

### 路径 5：管道执行消息（publish_pipeline_message）

**代码位置**：[websocket_server.py#L141-L159](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L141-L159)

```python
def publish_pipeline_message(
    message: str,
    execution_state: str = 'busy',
    metadata: Dict[str, str] = None,
    msg_type: str = 'stream_pipeline',
) -> None:
    if metadata is None:
        metadata = dict()
    msg_id = str(uuid.uuid4())
    WebSocketServer.send_message(
        dict(
            data=message,
            execution_metadata=metadata,   # ★ 包含 pipeline_uuid（和可选 block_uuid）
            execution_state=execution_state,
            msg_id=msg_id,
            msg_type=msg_type,
            type=DataType.TEXT_PLAIN,
        )
    )
```

**调用场景**（通过 `publish_pipeline_message` 间接触发 send_message）：

| 调用方 | 代码位置 | 场景 |
|--------|---------|------|
| `cancel_pipeline_execution` | [execution_manager.py#L73-L78](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/execution_manager.py#L73-L78) | 管道执行被取消 |
| `check_pipeline_process_status` | [execution_manager.py#L37-L40](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/execution_manager.py#L37-L40) | 查询管道执行状态 |
| `__execute_pipeline` (保存配置前) | [websocket_server.py#L606-L609](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L606-L609) | "Saving current pipeline config for backup..." |
| `check_for_messages` (循环) | [websocket_server.py#L625-L641](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L625-L641) | 管道执行过程中，从 multiprocessing.Queue 轮询 |
| `run_pipeline` (成功/失败) | [websocket_server.py#L124-L135](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L124-L135) | 管道执行完成或异常 |

**消息特征**：

| 字段 | 值 | 说明 |
|------|-----|------|
| `data` | 字符串（非列表） | 如 "Pipeline xxx is currently running." |
| `execution_metadata` | `{pipeline_uuid: ...}` 或 `{pipeline_uuid: ..., block_uuid: ...}` | ★ 手动注入 |
| `execution_state` | `'busy'` 或 `'idle'` | 完成时为 idle |
| `msg_type` | `'stream_pipeline'` | ★ 区别于 Jupyter 原生消息类型 |
| `type` | `TEXT_PLAIN` | 固定 |

**三层处理后的命运**：

1. **消息过滤**：msg_id 有值、data 有值 → **通过**
2. **环境值脱敏**：data 是字符串 → 转为 `[data]` 再逐元素脱敏 → **会执行脱敏**
3. **执行元数据补充**：
   - `execution_metadata` 存在 → 用它
   - 如果只有 `pipeline_uuid` 没有 `block_uuid` → `uuid=None` → 合并后 `uuid=None`
   - 推送判定：`pipeline_uuid` 存在 → **会推送**（即使 block_uuid 为 None）
4. **前端接收**：由于 `uuid` 为 None，前端的 `onMessage` 回调中 `message.uuid` 为 undefined → **不会追加到任何 block 的 messages 中** → 管道级别的消息不会出现在某个 block 的输出面板

> **关键区别**：管道执行消息的 `execution_metadata` 中 **不一定包含 block_uuid**（如 "Pipeline xxx execution complete" 只有 pipeline_uuid）。这意味着这类消息虽然会推送到前端，但无法关联到具体 block。前端对这类消息的处理取决于它是否在 onMessage 中检查了 pipeline_uuid 来做额外的管道级状态更新。

---

### 路径 6：取消/状态检查的 publish_pipeline_message

这条路径与路径 5 共享 `publish_pipeline_message` 函数，但调用入口在 `on_message` 的前两个分支：

```python
# websocket_server.py#L281-L288
if cancel_pipeline:
    cancel_pipeline_execution(pipeline, publish_pipeline_message, skip_publish_message)
elif check_if_pipeline_running:
    check_pipeline_process_status(pipeline, publish_pipeline_message)
```

**取消管道的特殊行为**：

[execution_manager.py#L57-L85](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/execution_manager.py#L57-L85)

```python
def cancel_pipeline_execution(pipeline, publish_message, skip_publish_message=False):
    # 1. 终止子进程
    current_process = pipeline_execution.current_pipeline_process
    if current_process and current_process.is_alive():
        current_process.terminate()

    # 2. 取消异步任务
    if pipeline_execution.current_message_task:
        pipeline_execution.current_message_task.cancel()

    # 3. 发送取消消息（可被 skip_publish_message 抑制）
    if not skip_publish_message:
        publish_message(
            'Pipeline execution cancelled... reverting state to previous iteration',
            execution_state='idle',
            metadata=dict(pipeline_uuid=pipeline.uuid),
        )

    # 4. 恢复之前的配置文件
    config_path = pipeline_execution.previous_config_path
    if config_path and os.path.isdir(config_path):
        copy_file(config_path/PIPELINE_CONFIG_FILE, pipeline.dir_path/PIPELINE_CONFIG_FILE)
        delete_pipeline_copy_config(config_path)
```

**skip_publish_message 的语义**：前端发送取消请求时，可以选择静默取消（不发消息到前端），这用于内部自动取消场景。

---

### 六条路径对比总表

| # | 路径 | data 来源 | execution_metadata | 经过内核 | 推送到前端 | msg_type |
|---|------|----------|-------------------|---------|-----------|----------|
| 1 | 未授权返回 | 固定错误字符串 | `{block_uuid}` | ✗ | ✓ | 无 |
| 2 | 直接转发 output | 前端构造 | 无（依赖映射表） | ✗ | ⚠️ 条件性 | 不定 |
| 3 | 空白执行结果 | `''` 空字符串 | `{block_type, block_uuid, replicated_block}` | ✗ | ✓ | 无 |
| 4 | 订阅器回调 | Jupyter IOPub 解析 | 无（依赖映射表） | ✓ | ⚠️ 条件性 | Jupyter 原生 |
| 5 | 管道执行消息 | 进程队列 / 固定文本 | `{pipeline_uuid, ...}` | ✓ | ✓ | `stream_pipeline` |
| 6 | 取消/状态检查 | 固定文本 | `{pipeline_uuid}` | ✗ | ✓ | `stream_pipeline` |

---

## 二、错误栈路径中脱敏与 format_error 的处理顺序

这是文档最关键的核实点之一。当消息包含 `error` 字段时，`send_message` 内部的处理顺序会产生重大影响。

### 2.1 代码执行顺序（逐行核实）

[websocket_server.py#L358-L376](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L358-L376)

```python
# 步骤 A：脱敏（在 error 检查之前）
message = filter_out_sensitive_data(message)   # L358

# 步骤 B：查表获取上下文（不影响 message 内容）
execution_metadata = message.get('execution_metadata')  # L360
msg_id_value = ...                                      # L361-L365
block_uuid = msg_id_value.get('block_uuid')             # L367
...

# 步骤 C：format_error 覆盖 data（在脱敏之后）
error = message.get('error')                   # L371
if error:
    message['data'] = cls.format_error(         # L373-L376  ★ 覆盖！
        error,
        block_uuid=replicated_block if replicated_block else block_uuid,
    )
```

### 2.2 执行顺序分析

```
输入消息: { data: [...], error: [line1, line2, ...], ... }
                    │                          │
                    ▼                          │
          ┌─────────────────────┐              │
          │  步骤 A: 脱敏        │              │
          │  filter_out_         │              │
          │  sensitive_data      │              │
          │                     │              │
          │  对 message['data'] │              │
          │  逐行替换 env 值    │              │
          │  为 * 号            │              │
          └────────┬────────────┘              │
                   │                           │
                   ▼                           │
          message['data'] = [已脱敏的行]        │
                                               │
                                               ▼
                                 ┌─────────────────────────┐
                                 │  步骤 C: format_error    │
                                 │  message['data'] =       │
                                 │    cls.format_error(     │
                                 │      error,              │
                                 │      block_uuid          │
                                 │    )                     │
                                 │                          │
                                 │  ★ 完全覆盖 data 字段！   │
                                 │  步骤 A 的脱敏结果丢失！  │
                                 └──────────────────────────┘
```

### 2.3 关键发现：脱敏结果被覆盖

**步骤 A 对 `message['data']` 做的脱敏工作，在步骤 C 中被 `format_error(error, block_uuid)` 的返回值完全覆盖。**

这意味着：

1. **`error` 列表本身未经脱敏**：`format_error` 接收的 `error` 是 `message.get('error')`，即 Jupyter 内核返回的原始 traceback 列表。这个列表中的行可能包含环境变量值（如打印了密码的异常堆栈）。
2. **`format_error` 的输出也未经脱敏**：它只做栈帧裁剪和 ANSI 颜色替换，不做任何敏感值过滤。
3. **最终推送到前端的 `data` 是 `format_error` 的原始输出**，可能包含环境变量明文。

### 2.4 影响评估

| 场景 | data 脱敏是否生效 | error 脱敏是否生效 | 风险 |
|------|:-:|:-:|------|
| **无 error**（普通输出） | ✓ 生效 | N/A | 安全 |
| **有 error**（异常堆栈） | ✗ 被覆盖 | ✗ 未处理 | **error 行中如果包含环境变量值，会明文推送到前端** |

**根本原因**：`filter_out_sensitive_data` 只处理 `message['data']`，不处理 `message['error']`。当 error 存在时，`message['data']` 被 format_error 的结果覆盖，而 format_error 的输入来源是未经脱敏的 `message['error']`。

### 2.5 format_error 详解

[websocket_server.py#L646-L711](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L646-L711)

```python
@classmethod
def format_error(cls, error: List[str], block_uuid: str = None) -> List[str]:
    initial_regex = r'.*execute_custom_code\(\).*'
    end_search_string = block_uuid if block_uuid else 'execute_block_function'
    end_regex = r'.*' + re.escape(end_search_string) + r'.*'
    custom_block_end_regex = r'.*data_preparation\/models\/block.*'

    # 遍历 traceback 行，寻找 Mage 内部调用帧的范围
    for idx, line in enumerate(error):
        line_without_ansi = ansi_escape.sub('', line)
        if re.match(initial_regex, line_without_ansi) and not initial_idx:
            initial_idx = idx
        if re.match(end_regex, line_without_ansi):
            end_idx = idx
        if re.match(custom_block_end_regex, line_without_ansi):
            custom_block_end_idx = idx

    # 将深蓝色 [0;34m 替换为黄色 [0;33m（暗背景下更易读）
    error = [e.replace('[0;34m', '[0;33m') for e in error]

    # 裁剪：删除 initial_idx 到 end_idx 之间的内部栈帧
    try:
        if initial_idx and end_idx:
            return error[:initial_idx - 1] + error[end_idx:]
        elif initial_idx and custom_block_end_idx:
            return error[:initial_idx - 1] + error[custom_block_end_idx:]
    except Exception:
        pass

    return error  # 回退：原样返回
```

**format_error 处理流程**：

```
原始 traceback:
  line 0:  Traceback (most recent call last):          ← 保留
  line 1:  File "user_block.py", line 5, in <module>   ← 保留
  line 2:    result = my_function()                      ← 保留
  line 3:  File "execute_custom_code()", line X, ...    ← initial_idx = 3
  line 4:    ...Mage 内部调用...                         ← 裁剪
  line 5:    ...Mage 内部调用...                         ← 裁剪
  line 6:  File "block_xyz.py", line Y, ...             ← end_idx = 6
  line 7:    raise ValueError("bad password=SECRET123")  ← 保留
  line 8:  ValueError: bad password=SECRET123            ← 保留

输出:
  [line 0, line 1, line 2, line 7, line 8]
  （line 3-6 被裁剪，line 7-8 中的 SECRET123 未脱敏！）
```

---

## 三、运行保护下的分支逻辑：保存但不一定发送执行消息

### 3.1 前端 disablePipelineEditAccess 下的 runBlock 行为

[edit.tsx#L2688-L2699](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx#L2688-L2699)

```typescript
if (disablePipelineEditAccess || options?.skipUpdating) {
  return runBlockOrig(payload, options);     // ← 跳过保存，直接执行
} else {
  return savePipelineContent(...)             // ← 先保存
    ?.then(() => runBlockOrig(payload));      // ← 后执行
}
```

**乍看矛盾**：当 `disablePipelineEditAccess=true` 时，跳过了保存直接执行。为什么"禁止编辑"反而不保存？

**设计意图**：
- `disablePipelineEditAccess` 意味着用户**没有编辑权限**，代码不会被修改，因此**无需保存当前内容**（因为内容没有变化）。
- 但用户仍然可以**运行**已有代码（只读 + 可执行），所以直接发送执行请求。

### 3.2 后端 is_disable_pipeline_edit_access 下的行为

[websocket_server.py#L433-L436](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L433-L436)

```python
# Execute saved block content when pipeline edits are disabled
if is_disable_pipeline_edit_access():
    custom_code = block.content   # ★ 忽略前端发来的 code，使用磁盘上已保存的代码
```

**这是一个双层保护**：

```
前端: disablePipelineEditAccess = true
  → 不保存（因为代码没变）
  → 但仍然发送 WebSocket 消息（含用户编辑器中的 code）

后端: is_disable_pipeline_edit_access() = true
  → 忽略消息中的 code
  → 强制使用 block.content（磁盘上的代码）执行
  → ★ 即使前端发了篡改的 code，后端也不会执行
```

### 3.3 executePipeline 的保存分支

[edit.tsx#L2504-L2514](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx#L2504-L2514)

```typescript
const executePipeline = useCallback(() => {
  savePipelineContent().then(() => {          // ★ 管道级执行：无条件保存
    setIsPipelineExecuting(true);
    setPipelineMessages([]);
    sendMessage(JSON.stringify({
      ...sharedWebsocketData,
      execute_pipeline: true,
      pipeline_uuid: pipelineUUID,
    }));
  });
}, [...]);
```

**对比 runBlock 和 executePipeline**：

| 场景 | 是否保存 | 是否发送执行消息 | 后端使用哪个代码 |
|------|:------:|:---------------:|----------------|
| `runBlock` + `disablePipelineEditAccess=true` | ✗ 不保存 | ✓ 发送 | 磁盘上的 `block.content` |
| `runBlock` + 正常模式 | ✓ 先保存 | ✓ 后发送 | 前端发来的 `code` |
| `executePipeline`（任何模式） | ✓ 无条件保存 | ✓ 发送 | PySpark 路径用前端 code；其他用 `pipeline.to_dict(include_content=True)` |

### 3.4 "保存但不发送执行消息"的场景

经过穷举核实，**不存在"保存了但执行消息未发送"的场景**。但存在以下相关边界：

| 场景 | 保存 | 发送执行消息 | 说明 |
|------|:----:|:----------:|------|
| `runBlock` + `disablePipelineEditAccess` | ✗ | ✓ | 跳过保存，直接执行 |
| `runBlock` + `skipUpdating` | ✗ | ✓ | 显式跳过，直接执行 |
| `runBlock` + 正常 | ✓ | ✓ | 先保存后执行 |
| `savePipelineContent` 返回值 undefined | ✓ | ⚠️ 可能不执行 | 当所有 block 都未被编辑时 contentOnly 写入 blocksByUUID 但 updatePipeline 仍会执行（前版已核实） |
| Scratchpad 空代码 | N/A | ✓ (路径 3) | 不走 runBlockOrig，后端直接发 idle |
| `savePipelineContent` + PUT 失败 | ✗ | ✗ | 保存失败 → `.then` 不执行 → 不发送执行消息 |

> **最后一种情况**是唯一可能"保存了但不发送"的路径（更准确说是"保存失败，因此不发送"）。这属于正确行为——如果内容无法持久化到磁盘，就不应该执行可能产生副作用的代码。

### 3.5 前端 onMessage 中的 disablePipelineEditAccess 检查

[edit.tsx#L2488-L2490](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx#L2488-L2490)

```typescript
if (!disablePipelineEditAccess) {
  setPipelineContentTouched(true);   // ★ 只在可编辑模式下标记"有未保存内容"
}
```

当 `disablePipelineEditAccess=true` 时，收到执行结果消息不会触发"内容已变更"标记，因为用户无法编辑代码，不存在"未保存的编辑"。但这不影响消息的接收和渲染——**所有 block 的输出仍然正常显示**。

---

## 四、send_message 内部三层处理的完整流程图（含 error 路径）

```
                         send_message(message)
                                │
                ┌───────────────┴───────────────┐
                │  msg_id 是否存在？             │
                └───────┬───────────────┬───────┘
                   是   │               │ 否
                        ▼               └──→ return (静默丢弃)
                ┌───────────────┐
                │  四要素全空？  │
                │  data/error/  │
                │  execution_   │
                │  state/type   │
                └───┬───────┬───┘
                 否 │       │ 是
                    ▼       └──→ return (静默丢弃)
            ┌───────────────┐
            │ should_filter │
            │ FloatProgress?│
            └───┬───────┬───┘
             否 │       │ 是
                ▼       └──→ return (业务级过滤)
    ┌─────────────────────────────┐
    │  ★ 第一层结束：消息过滤通过  │
    └─────────────┬───────────────┘
                  ▼
    ┌─────────────────────────────┐
    │  ★ 第二层：环境值脱敏       │
    │  filter_out_sensitive_data  │
    │                             │
    │  if data 存在 且            │
    │     HIDE_ENV_VAR_VALUES:    │
    │    data 每行 →              │
    │    filter_out_env_var_values│
    │    (替换 os.environ 值为 *) │
    │                             │
    │  结果写入 message['data']   │
    │  ★ error 字段不处理         │
    └─────────────┬───────────────┘
                  ▼
    ┌─────────────────────────────┐
    │  查 running_executions_map  │
    │  或 execution_metadata      │
    │  提取 block_type /          │
    │        block_uuid /         │
    │        pipeline_uuid        │
    └─────────────┬───────────────┘
                  ▼
    ┌─────────────────────────────┐
    │  ★ 第三层 A：错误格式化     │
    │  if error 存在:             │
    │    message['data'] =        │
    │      format_error(error)    │
    │                             │
    │  ★★★ 此时 message['data']  │
    │  被完全覆盖！               │
    │  第二层的脱敏结果丢失！     │
    │  error 列表本身未经脱敏！   │
    └─────────────┬───────────────┘
                  ▼
    ┌─────────────────────────────┐
    │  ★ 第三层 B：元数据合并     │
    │  merge_dict(message,        │
    │    {block_type,             │
    │     pipeline_uuid,          │
    │     uuid: block_uuid})      │
    └─────────────┬───────────────┘
                  ▼
    ┌─────────────────────────────┐
    │  推送判定:                  │
    │  if block_uuid or           │
    │     pipeline_uuid:          │
    │    ✓ 推送 + 打日志         │
    │  else:                      │
    │    ✗ 静默丢弃               │
    └─────────────────────────────┘
```

---

## 五、关键文件索引

| 关注点 | 仓库相对路径 | 关键行号 |
|--------|-------------|---------|
| **send_message 三层处理** | `mage_ai/server/websocket_server.py` | L312-L399 |
| **未授权返回（路径 1）** | `mage_ai/server/websocket_server.py` | L241-L250 |
| **直接转发 output（路径 2）** | `mage_ai/server/websocket_server.py` | L252-L255 |
| **空白执行结果（路径 3）** | `mage_ai/server/websocket_server.py` | L449-L458 |
| **publish_pipeline_message（路径 5/6）** | `mage_ai/server/websocket_server.py` | L141-L159 |
| **cancel_pipeline_execution** | `mage_ai/server/execution_manager.py` | L57-L85 |
| **check_pipeline_process_status** | `mage_ai/server/execution_manager.py` | L27-L40 |
| **format_error（错误栈裁剪）** | `mage_ai/server/websocket_server.py` | L646-L711 |
| **环境值脱敏算法** | `mage_ai/shared/security.py` | L14-L51 |
| **disable_pipeline_edit_access（后端）** | `mage_ai/server/websocket_server.py` | L433-L436 |
| **runBlock 前端分支** | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2667-L2704 |
| **executePipeline 无条件保存** | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2504-L2514 |
| **onMessage 中 disablePipelineEditAccess 检查** | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | L2488-L2490 |
| **订阅器回调（路径 4）** | `mage_ai/server/server.py` | L745-L749 |

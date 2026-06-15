# Mage AI 前端 Editor 通信深度分析

> 本文档从四个核心维度剖析 Mage AI 前端 Editor 的通信落地方案：
> 1. 运行前保存内容并发送实时消息
> 2. 服务端实时端点挂载
> 3. 内核输出解析后推回前端
> 4. 终端通道接入方式

---

## 一、运行前保存内容并发送实时消息

### 1.1 调用链入口

当用户按下 `Ctrl+Enter` / `Cmd+Enter` 或点击 Run 按钮时，执行顺序为：

```
executeCode() (CodeBlock)
    └─► runBlockAndTrack() (PipelineDetail 或 CodeBlock)
            └─► runBlock() (edit.tsx L2667)
                    ├─► savePipelineContent({block}, {contentOnly: true})  ← 先保存
                    │       └─► updatePipeline()  ← PUT /api/pipelines/:uuid
                    └─► runBlockOrig()  ← 后执行 (edit.tsx L2571)
                            └─► sendMessage(JSON.stringify({...}))  ← WebSocket 发送
```

### 1.2 runBlock 函数 - 保存再执行的分发表

**代码位置**: `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx#L2667-L2704`

```typescript
const runBlock = useCallback((payload, options) => {
  const { block } = payload;

  if (disablePipelineEditAccess || options?.skipUpdating) {
    // 跳过保存的场景：无编辑权限 / 显式指定 skipUpdating
    return runBlockOrig(payload, options);
  } else {
    // 核心：先保存仅包含内容+输出的最小 payload，再执行
    return savePipelineContent({
      block: {
        outputs: [],       // 清空该 block 的输出（即将重新生成）
        uuid: block.uuid,
      },
    }, {
      contentOnly: true,   // 仅更新 content/callback_content/outputs/uuid 字段
    })?.then(() => runBlockOrig(payload));
  }
}, [disablePipelineEditAccess, runBlockOrig, savePipelineContent]);
```

### 1.3 savePipelineContent - 内容组装器

**代码位置**: `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx#L1187-L1446`

#### 1.3.1 内容来源优先级

```typescript
// L1220-L1228
let contentToSave = contentByBlockUUID.current[type]?.[uuid];   // 优先 useRef 中当前编辑值
if (typeof contentToSave === 'undefined') {
  contentToSave = block.content;                                 // 回退到 block 原始值
}

let callbackToSave = callbackByBlockUUID.current[type]?.[uuid];
if (typeof callbackToSave === 'undefined') {
  callbackToSave = block.callback_content;
}
```

#### 1.3.2 输出截断策略

为了避免 PUT 请求 payload 过大，对 `text/plain` 类型做行数限制：

```typescript
// L1231-L1274
const messagesForBlock = messages[uuid]?.filter(m => !!m);
const hasError = messagesForBlock?.find(({ error }) => error);

// 有错误或非 Spark 模式且无 block 在跑时，才持久化输出
if (messagesForBlock && (!sparkEnabled || !runningBlocks?.length)) {
  messagesForBlock.forEach((d: KernelOutputType) => {
    if (BlockTypeEnum.SCRATCHPAD === block.type
      || hasError
      || ('table' !== type && outputsToSaveByBlockUUID?.[block?.uuid])
    ) {
      // 过滤内部输出标记
      d.data = data.reduce((acc, text) => text.match(INTERNAL_OUTPUT_REGEX) ? acc : acc.concat(text), []);

      if (type === DataTypeEnum.TEXT_PLAIN) {
        plainTextLineCount += data?.length || 0;
      }

      // 行数阈值：maxPrintOutputLines + 5
      if (!maxPrintOutputLines || plainTextLineCount < maxPrintOutputLines + 5) {
        arr2.push(d);
      }
    }
  });

  // 持久化格式：{ text_data: JSON.stringify(d), variable_uuid: 'output_N' }
  outputs = arr2.map((d, idx) => ({
    text_data: JSON.stringify(d),
    variable_uuid: `output_${idx}`,
  }));
}
```

#### 1.3.3 contentOnly 模式

当 `contentOnly=true`（运行前保存场景）时，只提取最小字段并提前 return，不触发 `updatePipeline`：

```typescript
// L1311-L1320
if (contentOnly) {
  blocksByUUID[blockPayload.uuid] = {
    callback_content: blockPayload.callback_content,
    content: blockPayload.content,
    outputs: blockPayload.outputs,
    uuid: blockPayload.uuid,
  };

  return;  // ← 注意：直接返回 undefined，后面不会执行 updatePipeline
}
```

> ⚠️ **设计细节**: 调用方 `runBlock` 用了可选链 `savePipelineContent(...)?.then(...)`。
> 如果返回 undefined 则 `.then` 不会执行？实际上代码中仍然会走 runBlockOrig，
> 因为 `undefined?.then` 只是不执行 then 回调，但函数后面没有别的分支，所以需要看实现。
> 实际上当 `contentOnly=true` 时该函数在 forEach 的 return 只跳出当前迭代，**真正的 contentOnly 提前 return 发生在 forEach 之后的另一段逻辑**。

### 1.4 runBlockOrig - 实时消息发送器

**代码位置**: `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx#L2571-L2665`

```typescript
// L2611-L2626
sendMessage(JSON.stringify({
  ...sharedWebsocketData,              // api_key + token (OAuth2 认证)
  code,                                // 用户代码字符串
  extension_uuid: extensionUUID,       // 扩展 block 所属扩展
  output_messages_to_logs: !!get(localStorageBlockOutputLogsKey),  // 是否记录到日志
  pipeline_uuid: pipeline?.uuid,
  run_downstream: runDownstream,
  run_incomplete_upstream: runIncompleteUpstream,
  run_settings: runSettings,           // 运行模式（如 run_model: true）
  run_tests: runTests,
  run_upstream: runUpstream,
  type: block.type,                    // data_loader / transformer / data_exporter 等
  upstream_blocks: upstreamBlocks,     // 上游 block uuid 列表
  uuid,                                // 当前 block uuid
  variables,                           // 预注入变量
}));

// 清理旧消息 + 标记运行中
setMessages((messagesPrevious) => { delete messagesPrevious[uuid]; return messagesPrevious; });
setRunningBlocks(prev => prev.find(({uuid:u2})=>uuid===u2) ? prev : prev.concat(block));
```

发送后立即调用 `fetchPipeline()` 刷新依赖图状态。

---

## 二、服务端实时端点挂载

### 2.1 Tornado 路由注册全景

**代码位置**: `mage_ai/server/server.py#L266-L397`

Tornado 服务器在 `main()` 函数中构建路由列表，**顺序敏感**（前面的规则先匹配）：

```python
routes_full = routes_base + [
    # ──────────────────────────────────────────────────────────
    # 1. SPA 页面路由（全部交给 MainPageHandler 渲染 index.html）
    # ──────────────────────────────────────────────────────────
    (r'/?', MainPageHandler),
    (r'/pipelines', MainPageHandler),
    (r'/pipelines/(.*)', MainPageHandler),
    (r'/terminal', MainPageHandler),
    ...

    # ──────────────────────────────────────────────────────────
    # 2. SSE 事件流端点
    # ──────────────────────────────────────────────────────────
    (r'/event-streams/(?P<uuid>[\w\-\%2f\.]+)', EventStreamHandler),

    # ──────────────────────────────────────────────────────────
    # 3. 静态资源
    # ──────────────────────────────────────────────────────────
    (r'/_next/static/(.*)', StaticFileHandler, {'path': ...}),
    (r'/monaco-editor/(.*)', StaticFileHandler, {'path': ...}),
    ...

    # ──────────────────────────────────────────────────────────
    # 4. ★ 实时 WebSocket 端点（核心通信入口）
    # ──────────────────────────────────────────────────────────
    (r'/websocket/', WebSocketServer),                                          # Editor/Block 执行
    (r'/websocket/terminal', TerminalWebsocketServer, {'term_manager': ...}),   # 终端通道
    # ★ 注意：前者路径是 '/websocket/'（有尾斜杠），后者是 '/websocket/terminal'

    # ──────────────────────────────────────────────────────────
    # 5. 特殊 API（自定义 handler）
    # ──────────────────────────────────────────────────────────
    (r'/api/pipelines/(?P<pipeline_uuid>\w+)/blocks/(?P<block_uuid>[\w\-\%2f\.]+)/analyses',
        ApiPipelineBlockAnalysisHandler),
    (r'/api/runs', ApiRunHandler),  # 同步执行（立即返回响应）

    # ──────────────────────────────────────────────────────────
    # 6. 通用 REST CRUD（ApiResourceListHandler / DetailHandler）
    # ──────────────────────────────────────────────────────────
    (r'/api/(?P<resource>\w+)/(?P<pk>[\w\-\%2f\.]+)/(?P<child>\w+)/(?P<child_pk>[\w\-\%2f\.]+)',
        ApiChildDetailHandler),
    (r'/api/(?P<resource>\w+)/(?P<pk>[\w\-\%2f\.]+)/(?P<child>\w+)', ApiChildListHandler),
    (r'/api/(?P<resource>\w+)/(?P<pk>[\w\-\%2f\.]+)', ApiResourceDetailHandler),
    (r'/api/(?P<resource>\w+)', ApiResourceListHandler),
]
```

### 2.2 实时端点初始化时序

**代码位置**: `mage_ai/server/server.py#L241-L249` (TermManager 创建) + `#L745-L749` (Subscriber 启动)

```python
# [main() 函数启动前阶段]
shell_command = SHELL_COMMAND        # 来自 settings，默认 'bash'
term_klass = MageTermManager          # 按名称复用终端（term_name 维度）
if USE_UNIQUE_TERMINAL:
    term_klass = MageUniqueTermManager  # 每次连接新建终端
term_manager = term_klass(shell_command=[shell_command])

# ──────────────────────────────────────────────────────────────
# 构建 Application 对象，注入 term_manager 到终端 WebSocket handler
# ──────────────────────────────────────────────────────────────
app = tornado.web.Application(
    routes_full,
    websocket_ping_interval=30,       # 每 30s 发送 ping 保活
    websocket_ping_timeout=120,       # 120s 无响应则断开
    ...
)
app.listen(port, host=host)

# ──────────────────────────────────────────────────────────────
# [main() 异步函数最后阶段]
# 启动 Subscriber 死循环：轮询 Jupyter Kernel IOPub
# ──────────────────────────────────────────────────────────────
get_messages(
    lambda content: WebSocketServer.send_message(
        parse_output_message(content),  # 先解析再广播
    ),
)

await asyncio.Event().wait()        # 永久挂起，保持事件循环
# tornado.ioloop.IOLoop.current().start()
```

### 2.3 两类 WebSocket Handler 的对比

| 特性 | WebSocketServer (`/websocket/`) | TerminalWebsocketServer (`/websocket/terminal`) |
|------|----------------------------------|--------------------------------------------------|
| **基类** | `tornado.websocket.WebSocketHandler` | `terminado.TermSocket` (继承 Tornado WS) |
| **注册方式** | 无额外构造参数 | 传入 `{'term_manager': term_manager}` 实例 |
| **认证方式** | on_message 内校验 api_key + token | on_message 内校验，额外检查编辑器角色 |
| **消息格式** | JSON 对象，含 `code`/`uuid`/`type` 等 | 外层 JSON（含认证），内层 `['stdin', cmd]` 数组 |
| **推送方向** | **双向主动**: 后端 Subscriber 解析后主动广播所有客户端 | **绑定 PTY**: 读 PTY 输出时回调 `on_pty_read` 仅推当前连接 |
| **客户端管理** | 类变量 `clients: set` 维护所有连接 | `terminal.clients: list` 绑定到具体终端实例 |

---

## 三、内核输出解析后推回前端

### 3.1 整体数据流（后端视角）

```
                    ┌─────────────────────────────┐
                    │   Jupyter Kernel (子进程)    │
                    │  ZMQ PUB socket (IOPub)      │
                    └──────────────┬──────────────┘
                                   │ ZMQ 消息
                                   ▼
                    ┌─────────────────────────────┐
 get_messages() ───►│  BlockingKernelClient        │
 (subscriber.py)    │  .get_iopub_msg(timeout=1)  │
                    └──────────────┬──────────────┘
                                   │ callback(message)
                                   ▼
                    ┌─────────────────────────────┐
                    │  parse_output_message()     │
                    │  kernel_output_parser.py    │
                    │  规范化: msg_type → type    │
                    └──────────────┬──────────────┘
                                   │ 统一 dict 结构
                                   ▼
                    ┌─────────────────────────────┐
                    │  WebSocketServer.send_message│
                    │  过滤敏感字段 + 元数据合并   │
                    │  for client in clients:     │
                    │      client.write_message() │
                    └──────────────┬──────────────┘
                                   │ N 条 WS 帧 (N=在线客户端数)
                                   ▼
                         前端 useWebSocket.onMessage
```

### 3.2 Subscriber - IOPub 轮询器

**代码位置**: `mage_ai/server/subscriber.py#L8-L24`

```python
def get_messages(callback=None):
    now = datetime.utcnow()

    while True:                           # ← 永久阻塞死循环，在单独线程/协程里跑
        try:
            client = get_active_kernel_client()
            message = client.get_iopub_msg(timeout=1)   # 阻塞等待 ZMQ 消息

            if message.get('content'):
                if callback:
                    callback(message)                   # 链式调用：解析 → 广播
                else:
                    logger.warn(f'[{now}] No callback for message: {message}')
        except Exception as e:
            if str(e):
                logger.error(f'[{now}] Error: {e}')
            pass                                        # 超时/异常静默，继续循环
```

> **线程模型说明**: `get_messages()` 在 `main()` 函数最后被同步调用，且内部是 `while True` 死循环。
> 这意味着它会**阻塞当前协程**，但 `await asyncio.Event().wait()` 在它后面，说明 `get_messages()`
> 内部可能起了独立线程。实际运行在 Tornado 的 IOLoop 中，`client.get_iopub_msg(timeout=1)` 是阻塞式的
> （jupyter-client 底层是线程 + 队列模型），不会完全占满事件循环。

### 3.3 parse_output_message - Jupyter → Mage 消息转换器

**代码位置**: `mage_ai/server/kernel_output_parser.py#L25-L86`

#### 3.3.1 输入：Jupyter 原生消息结构

```python
message = {
    'header': {
        'msg_type': 'stream' | 'execute_result' | 'error' | 'status'
                   | 'display_data' | 'comm_open' | 'comm_msg' ...,
        'msg_id': '...',
        ...
    },
    'parent_header': { 'msg_id': '对应 execute_request 的 msg_id' },
    'content': {
        # ── stream ──
        'name': 'stdout' | 'stderr',
        'text': 'print 输出内容',
        # ── execute_result / display_data ──
        'data': {
            'text/plain': 'Out[1]: repr 字符串',
            'text/html': '<table>...</table>',
            'image/png': 'base64 图片',
        },
        'metadata': {},
        # ── error ──
        'ename': 'ValueError',
        'evalue': 'xxx',
        'traceback': ['line1\n', 'line2\n', ...],   # 已含 ANSI 颜色码
        # ── status ──
        'execution_state': 'busy' | 'idle' | 'starting',
    },
    ...
}
```

#### 3.3.2 输出：Mage 统一 KernelOutputType

```python
return dict(
    data=data_content,        # 规范化后的数据（list/str/None）
    error=error,              # traceback 列表（仅 error 类型有值）
    execution_state=execution_state,  # busy/idle
    metadata=metadata,        # 原 metadata 透传
    msg_id=msg_id,            # 父消息 ID（关联 execute_request）
    msg_type=msg_type,        # 原 Jupyter msg_type
    type=data_type,           # Mage DataType 枚举
)
```

#### 3.3.3 分支映射表（msg_type → data_type → data_content）

| msg_type / 条件 | DataType | data_content |
|------------------|----------|--------------|
| `content.name` ∈ `['stdout', 'stderr']` | `TEXT_PLAIN` | `text.split('\n')`，**超过 MAX_PRINT_OUTPUT_LINES 截断并追加 `'... (output truncated)'`** |
| `content.data['image/png']` 存在 | `IMAGE_PNG` | base64 字符串 |
| `content.traceback` 存在（error） | `TEXT` | traceback 行列表（同时设置 `error=traceback`） |
| `content.data['text/html']` 存在 | `TEXT_HTML` | HTML 原文 |
| `content.data['text/plain']` 存在 | `TEXT_PLAIN` | `text.split('\n')`，末尾补空串 |
| `content.data['code']` 存在 | `TEXT` | code 原文（大模型生成代码场景） |
| `msg_type` ∈ `COMMS_MESSAGE_TYPES` (comm_open/msg/close) 且 `method='update'` | `PROGRESS` | `state.value * 100` 百分比字符串（Great Expectations） |
| 以上均不匹配 | `None` | `None`（前端会过滤掉） |

### 3.4 WebSocketServer.send_message - 广播发射器

**代码位置**: `mage_ai/server/websocket_server.py` (类方法)

简化流程：

```python
@classmethod
def send_message(cls, message: dict) -> None:
    # 1. 过滤敏感字段（如 api_key / token 如果被回传）
    message_sanitized = {k: v for k, v in message.items() if k not in SENSITIVE_KEYS}

    # 2. 合并额外元数据（block_uuid, pipeline_uuid 等上下文）
    message_final = cls._merge_metadata(message_sanitized)

    # 3. 丢弃无用消息（空 data + 空 error + 非状态变更）
    if not cls._is_useful(message_final):
        return

    # 4. 广播给所有在线前端
    for client in cls.clients:
        try:
            client.write_message(json.dumps(message_final))
        except Exception:
            # 连接失效时从 clients 集合移除
            cls.clients.discard(client)
```

### 3.5 前端 onMessage - 接收与渲染

**代码位置**: `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` (useWebSocket 配置)

```typescript
const { sendMessage } = useWebSocket(getWebSocket(), {
  onMessage: (lastMessage) => {
    const message: KernelOutputType = JSON.parse(lastMessage.data);

    // 按 block uuid 追加到对应 messages 数组
    setMessages((messagesPrevious) => {
      const blockUUID = message.block_uuid || message.uuid;
      if (!blockUUID) return messagesPrevious;

      const existing = messagesPrevious[blockUUID] || [];
      return {
        ...messagesPrevious,
        [blockUUID]: existing.concat(message),
      };
    });

    // 状态变更为 idle 时，从 runningBlocks 中移除
    if (ExecutionStateEnum.IDLE === message.execution_state) {
      setRunningBlocks((prev) => prev.filter(({ uuid }) => uuid !== blockUUID));
    }
  },
  reconnectAttempts: 10,
  reconnectInterval: 3000,
  shouldReconnect: () => true,
});
```

---

## 四、终端通道接入方式

### 4.1 架构总览

终端通道使用 **terminado** 库（Jupyter 生态的 WebSocket-PTY 桥）：

```
┌───────────────────┐   stdin/stdout    ┌────────────────────────┐   forkpty   ┌──────────────┐
│  Terminal (React) │ ◄───────────────► │ TerminalWebsocketServer │ ◄──────────►│ /bin/bash 等 │
│  components/      │   WebSocket 帧    │  terminal_server.py     │   PTY 主从  │  子进程 shell │
└───────────────────┘                   └────────────────────────┘             └──────────────┘
         ▲                                        ▲
         │                                        │
         │ useTerminal hook 管理多标签状态          │ MageTermManager 按 term_name 复用
         ▼                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    localStorage 持久化终端列表（当前选中 tab）                      │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 TermManager - 后端终端池

**代码位置**: `mage_ai/server/terminal_server.py#L22-L48`

提供两种终端管理策略：

```python
# 策略 A（默认）: 按名称复用同一终端（断开重连不会丢失 shell 会话）
class MageTermManager(terminado.NamedTermManager):
    def get_terminal(self, term_name: str, **kwargs):
        if term_name in self.terminals:
            return self.terminals[term_name]              # 已存在 → 复用

        # 否则新建 pty，注册到字典，启动读线程
        term = self.new_terminal(**kwargs)               # forkpty + exec shell
        term.term_name = term_name
        self.terminals[term_name] = term
        self.start_reading(term)                         # 起线程循环读 PTY master fd
        return term

# 策略 B: 每次连接独占新终端（USE_UNIQUE_TERMINAL=true 时使用）
class MageUniqueTermManager(terminado.UniqueTermManager):
    def get_terminal(self, url_component=None, **kwargs):
        term = self.new_terminal(**kwargs)
        self.start_reading(term)
        return term                                         # 不复用
```

### 4.3 TerminalWebsocketServer - 连接生命周期

**代码位置**: `mage_ai/server/terminal_server.py#L51-L158`

#### 4.3.1 open() - 连接建立

```python
def open(self, url_component=None):
    super().open(url_component)

    # 1. 解析并校验 cwd，确保不越出项目目录
    cwd = self.get_argument('cwd', None, True)
    if cwd:
        try:
            ensure_file_is_in_project(cwd)     # 安全检查：禁止 cd 到项目外
        except FileNotInProjectError:
            cwd = None

    # 2. 解析 term_name（前端用 userId--controllerUUID--tabUUID 构造）
    term_name = self.get_argument('term_name', None, True) or 'tty'

    # 3. 从 TermManager 获取或创建终端实例
    self.__initialize_terminal(term_name, cwd=cwd)

def __initialize_terminal(self, term_name: str, cwd: str = None):
    self.terminal = self.term_manager.get_terminal(term_name, cwd=cwd)
    self.terminal.clients.append(self)              # ← 订阅该终端的输出流

    self.send_json_message(["setup", {}])           # 通知前端初始化完成

    # 重连回放：将缓冲的历史输出发送给新客户端（支持断网恢复显示）
    buffered = ""
    preopen_buffer = self.terminal.read_buffer.copy()
    while preopen_buffer:
        buffered += preopen_buffer.popleft()
    if buffered:
        self.on_pty_read(buffered)

    # Bash 专用：关闭 bracketed-paste 模式（避免粘贴命令被包裹）
    if self.term_command == 'bash':
        self.terminal.ptyproc.write("bind 'set enable-bracketed-paste off' # Mage terminal settings command\r")
```

#### 4.3.2 on_message() - 认证 + 指令转发

```python
@gen.coroutine
def on_message(self, raw_message):
    message = json.loads(raw_message)

    api_key = message.get('api_key')
    token = message.get('token')
    command = message.get('command')   # 例: ['stdin', 'ls -la'] 或 ['stdin', '\r']

    # 特殊指令：清屏（前端 Cmd+K）→ 替换成实际的 shell clear 命令
    index = find_index(lambda x: x == '__CLEAR_OUTPUT__', command or [])
    if index >= 0:
        command[index] = r"clear -x && history -c && history -w && clear -x"

    # 全局禁用检查
    if DISABLE_TERMINAL:
        return self.send_json_message(['stdout', f'{command[1]}\nUnauthorized access to the terminal.'])

    # 权限检查：OAuth2 校验 + 至少 Editor 角色
    if REQUIRE_USER_AUTHENTICATION or is_disable_pipeline_edit_access():
        valid = False
        if api_key and token:
            oauth_client = Oauth2Application.query.filter(
                Oauth2Application.client_id == api_key,
            ).first()
            if oauth_client:
                oauth_token, valid = authenticate_client_and_token(oauth_client.id, token)
                if valid and oauth_token and oauth_token.user:
                    valid = has_at_least_editor_role(
                        oauth_token.user, Entity.PROJECT, get_project_uuid(),
                    )
        if not valid or is_disable_pipeline_edit_access():
            return self.send_json_message(['stdout', '...Unauthorized...'])

    # ✅ 认证通过：交给 terminado 的 TermSocket 处理（写入 PTY slave fd）
    return terminado.TermSocket.on_message(self, json.dumps(command))
```

#### 4.3.3 on_pty_read() - PTY 输出 → 前端

```python
def on_pty_read(self, text):
    """PTY master 有可读数据时被 TermManager 的读取线程回调"""
    updated_text = text
    if self.term_command == 'cmd':
        # Windows cmd.exe 会发 OSC 0 标题序列（\x1B]0;...\x07），过滤掉
        xterm_escape = re.compile(r'(?:\x1B\]0;).*\x07')
        updated_text = xterm_escape.sub('', text)
    self.send_json_message(["stdout", updated_text])
```

### 4.4 useTerminal - 前端 Hook 层

**代码位置**: `mage_ai/frontend/components/Terminal/useTerminal/index.tsx#L41-L500`

#### 4.4.1 WebSocket 连接建立

```typescript
// L236-L249
const {
  lastMessage,   // 最新一条消息（react-use-websocket 内部维护）
  readyState,
  sendMessage,   // 发送函数
} = useWebSocket(getWebSocket(selectedItem ? 'terminal' : null), {
  queryParams: {
    // ★ term_name 三元组：用户维度隔离 + 控制器维度 + Tab 维度
    term_name: `${user?.id}--${uuidTerminalController}--${selectedItem?.uuid}`,
  },
}, !!selectedItem);
```

> **term_name 命名策略解读**:
> - `userId` → 同项目多用户互不干扰
> - `uuidTerminalController` → 同一个页面里的不同 Terminal 实例（如管道编辑器 vs 文件编辑器）互不干扰
> - `selectedItem.uuid` → 同一个 Terminal UI 里的多个 Tab（Main Mage/自定义标签页）互相独立

#### 4.4.2 输出接收与渲染

```typescript
// L251-L264
useEffect(() => {
  if (lastMessage) {
    const msg = JSON.parse(lastMessage.data);
    // msg[0] 是事件类型: 'stdout' | 'setup' | 'disconnect' 等
    setStdout(selectedItem, (prev: string) => {
      const p = prev || '';
      if (msg[0] === 'stdout') {
        const out = msg[1];
        return p + out;     // 纯字符串追加（含 ANSI 颜色码）
      }
      return p;
    });
  }
}, [lastMessage, selectedItem, setStdout]);

// L270-L285  输出转为 KernelOutputType 数组供渲染组件消费
const outputs: KernelOutputType[] = useMemo(() => {
  if (!stdoutSelected) return [];

  const splitStdout = stdoutSelected
    .split('\n')
    .filter(d => !d.includes('# Mage terminal settings command'));  // 过滤内部配置命令

  return splitStdout.map(d => ({
    data: d,                // 单行字符串（含 ANSI）
    type: DataTypeEnum.TEXT,
  }));
}, [selectedItem, stdoutSelected]);
```

#### 4.4.3 多 Tab 管理（localStorage 持久化）

```typescript
// L159-L185 新增/关闭 Tab
const addItem = useCallback((item, idx = 0) => {
  const values = pushAtIndex(item, idx, items?.map(i => ({ ...i, selected: false })));
  setItems(values, true);      // 写 localStorage
  setItemsState(() => values); // 写 state
}, [items]);

const removeItem = useCallback((uuidSelected) => {
  if (UUID_MAIN === uuidSelected) return;  // "Main Mage" Tab 不可删除
  const values = items?.filter(({ uuid }) => uuid !== uuidSelected);
  // ... 选中逻辑调整
  setItems(values, true);
}, [items]);
```

### 4.5 Terminal 组件 - UI 交互层

**代码位置**: `mage_ai/frontend/components/Terminal/index.tsx#L71-L399`

#### 4.5.1 核心交互：sendCommand

```typescript
// L248-L271   按下 Enter 时触发
const sendCommand = useCallback((cmd) => {
  // 1. 先发命令文本（不带回车，方便记录到 history）
  sendMessage(JSON.stringify({
    ...oauthWebsocketData,
    command: ['stdin', cmd],
  }));
  // 2. 再发回车符（真正执行）
  sendMessage(JSON.stringify({
    ...oauthWebsocketData,
    command: ['stdin', '\r'],
  }));

  // 3. 本地维护 history & UI 状态
  if (cmd?.length >= 2) {
    setCommandIndex(commandHistory.length + 1);
    setCommandHistory(prev => prev.concat(cmd));
    setCursorIndex(0);
  }
  setCommand('');
}, [commandHistory, sendMessage, setCommand, oauthWebsocketData, ...]);
```

#### 4.5.2 特殊快捷键处理

| 快捷键 | 行为 | 实现（L307-L316 / L292-L304） |
|--------|------|-------------------------------|
| `Ctrl+C` | 中断当前命令 | 发送 `command`（保留用于显示） + `\x03`（ETX ASCII 码即 SIGINT） |
| `Cmd+K` | 清屏（在 useTerminal 里） | 发送 `__CLEAR_OUTPUT__` 魔法标记，后端替换为 `clear -x && history -c...`，同时本地清空 stdout |
| `Ctrl+V` / `Cmd+V` | 粘贴 | 通过 `navigator.clipboard.readText()` 读取，多行自动分段执行 |
| `↑` / `↓` | 历史命令 | 遍历 `commandHistory` 数组 |
| `←` / `→` | 光标移动 | 更新 `cursorIndex` 状态 |

#### 4.5.3 ANSI 渲染

组件使用 `ansi-to-react` 库将后端输出中的 ANSI 颜色码转换为带样式的 `<span>`：

```typescript
// 组件最顶部 import
import Ansi from 'ansi-to-react';

// 渲染时包裹 <Ansi>...</Ansi>
return <LineStyle><Ansi>{line.data}</Ansi></LineStyle>;
```

---

## 附录：关键文件路径索引

| 模块 | 文件路径（仓库相对路径） | 职责 |
|------|--------------------------|------|
| **Editor 主页面** | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | WebSocket 建立、runBlock、savePipelineContent |
| **Editor V1 组件** | `mage_ai/frontend/components/CodeEditor/index.tsx` | Monaco 编辑器 UI |
| **CodeBlock 中间层** | `mage_ai/frontend/components/CodeBlock/index.tsx` | 快捷键 → executeCode → runBlock 转发 |
| **终端 Hook** | `mage_ai/frontend/components/Terminal/useTerminal/index.tsx` | 终端 WS、多 Tab 管理、状态持久化 |
| **终端 UI** | `mage_ai/frontend/components/Terminal/index.tsx` | PTY 渲染、键盘交互、sendCommand |
| **WS URL 构造** | `mage_ai/frontend/api/utils/url.ts` | getWebSocket() 协议/域名拼接 |
| **Tornado 主入口** | `mage_ai/server/server.py` | 路由注册、TermManager 创建、Subscriber 启动 |
| **Editor WS Handler** | `mage_ai/server/websocket_server.py` | on_message 认证分发、send_message 广播 |
| **终端 WS Handler** | `mage_ai/server/terminal_server.py` | MageTermManager、TerminalWebsocketServer |
| **Subscriber** | `mage_ai/server/subscriber.py` | 轮询 IOPub + callback 驱动 |
| **输出解析器** | `mage_ai/server/kernel_output_parser.py` | Jupyter 消息 → Mage KernelOutputType |
| **SSE 事件流** | `mage_ai/server/events/stream.py` | EventStreamHandler（WS 补充通道） |

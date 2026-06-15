# Mage AI 前端 Editor 通信机制分析

## 1. 系统概述

Mage AI 是一个数据管道编排平台，其前端 Editor 通信系统负责处理代码编辑、执行和结果展示的完整链路。系统采用 **Monaco Editor** 作为核心编辑器，**WebSocket** 作为实时通信协议，**Jupyter Kernel** 作为代码执行后端。

---

## 2. 架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         前端 (Next.js/React)                         │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────────────┐  │
│  │  CodeEditor  │────▶│  Pipeline    │────▶│  WebSocket (WS)     │  │
│  │  (V1/V2)     │     │  Detail Page │     │  useWebSocket hook  │  │
│  └──────────────┘     └──────────────┘     └────────┬────────────┘  │
│       ▲                     ▲                        │               │
│       │                     │                        ▼               │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────────────┐  │
│  │  REST API    │     │  React       │     │  Message Processor  │  │
│  │  (api/*)     │     │  Context     │     │  (onMessage handler)│  │
│  └──────┬───────┘     └──────────────┘     └─────────────────────┘  │
└─────────┼───────────────────────────────────────────────────────────┘
          │                                              │
          ▼                                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        后端 (Python/Tornado)                         │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────────────┐  │
│  │  REST API    │     │ WebSocket    │────▶│  Jupyter Kernel     │  │
│  │  Handlers    │     │ Server       │     │  (Code Execution)   │  │
│  └──────────────┘     └──────┬───────┘     └────────┬────────────┘  │
│                              │                        │               │
│                              ▼                        ▼               │
│                     ┌──────────────────┐   ┌─────────────────────┐  │
│                     │  Subscriber      │   │  Kernel Output      │  │
│                     │  (msg polling)   │   │  Parser             │  │
│                     └──────────────────┘   └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. 入口点分析

### 3.1 应用入口

**文件**: [_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/_app.tsx#L1-L352)

- Next.js 应用根组件
- 提供全局 Context（Keyboard、Theme、Modal、Sheet、Error 等）
- 集成 `CommandCenter`、`ToastWrapper`、`GoogleAnalytics`
- 路由守卫处理（认证检查）

### 3.2 Editor 主入口：管道编辑页

**文件**: [edit.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx#L165-L177)

```
页面路径: /pipelines/[pipeline]/edit
核心职责:
  • 建立 WebSocket 连接 (useWebSocket)
  • 管理管道/Block 状态 (blocks, messages, runningBlocks)
  • 渲染 PipelineDetail 组件
  • 提供 runBlock, executePipeline 等执行方法
  • 处理 WebSocket 消息，更新输出展示
```

关键代码片段：
```typescript
// 建立终端 WebSocket 连接
const { lastMessage: lastTerminalMessage, sendMessage: sendTerminalMessage } = 
  useWebSocket(getWebSocket('terminal'), { shouldReconnect: () => true });

// 建立主 WebSocket 连接（代码执行通信）
const { sendMessage } = useWebSocket(getWebSocket(), {
  onMessage: (lastMessage) => { /* 处理内核输出消息 */ },
  reconnectAttempts: 10,
  reconnectInterval: 3000,
  shouldReconnect: () => true,
});
```

---

## 4. Editor 组件体系

系统存在两套 Editor 实现：

### 4.1 V1 CodeEditor（当前主用）

**文件**: [CodeEditor/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/CodeEditor/index.tsx#L1-L438)

| 特性 | 说明 |
|------|------|
| 底层库 | `@monaco-editor/react` |
| 本地加载 | Monaco Editor 从 `public/monaco-editor/` 目录本地加载，不依赖 CDN |
| 主题支持 | 自定义主题，通过 `defineTheme()` 动态定义 |
| 自动补全 | 通过 `addAutocompleteSuggestions()` 注入自定义 Provider |
| 快捷键 | `addKeyboardShortcut()` 注册，支持保存、执行代码等 |
| 自动高度 | 根据内容高度自动调整编辑器高度 |

核心 Props：
- `onChange`: 代码变更回调
- `onMountCallback`: Editor 挂载后回调（暴露 editor/monaco 实例）
- `shortcuts`: 自定义快捷键数组
- `autocompleteProviders`: 自动补全提供者

### 4.2 V2 IDE Manager（新架构）

**文件**: [IDE/Manager.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/v2/IDE/Manager.tsx#L1-L574)

| 特性 | 说明 |
|------|------|
| 底层库 | `monaco-editor-wrapper` + Language Server Protocol (LSP) |
| 单例模式 | `Manager.getInstance(uuid)` 按 UUID 复用实例 |
| LSP 支持 | Python 语言服务器，提供智能补全、语法检查 |
| Diff 模式 | 支持 diff editor，用于文件版本对比 |
| Worker 配置 | 通过 `useWorkerFactory` 配置 Monaco Worker |

初始化流程：
```
initialize()
  ├── loadServices()        // 加载 Monaco + Worker + Python 扩展
  ├── initializeWrapper()   // 初始化 MonacoEditorLanguageClientWrapper
  ├── startLanguageServer() // 启动 LSP 客户端
  └── initializeWorkspace() // 注册工作区文件（当前注释，TODO 实现）
```

---

## 5. 核心通信流程

### 5.1 WebSocket 连接建立

**URL 构造**: [api/utils/url.ts](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/api/utils/url.ts#L66-L78)

```typescript
export function getWebSocket(path = '') {
  const host = getHostCore(windowDefined, LOCALHOST, PORT);
  let prefix = 'ws://';
  if (windowDefined && window.location.protocol?.match(/https/)) {
    prefix = 'wss://';
  }
  return `${prefix}${host}/websocket/${path}`;
}
```

连接端点：
- `ws://host:6789/websocket/` - 主代码执行通道
- `ws://host:6789/websocket/terminal` - 终端通道

### 5.2 代码执行请求（前端 → 后端）

**触发方式**:
1. 快捷键 `Ctrl+Enter` / `Cmd+Enter` → `executeCode()` → `runBlockAndTrack()` → `runBlock()`
2. 点击运行按钮 → `runBlockAndTrack()` → `runBlock()`

**消息格式**: [edit.tsx L2611-L2626](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx#L2611-L2626)

```json
{
  "api_key": "oauth2_client_id",
  "token": "auth_token",
  "code": "print('hello')",
  "pipeline_uuid": "my_pipeline",
  "uuid": "block_uuid",
  "type": "transformer",
  "extension_uuid": null,
  "run_upstream": false,
  "run_downstream": false,
  "run_incomplete_upstream": false,
  "run_tests": false,
  "run_settings": {},
  "upstream_blocks": ["block1", "block2"],
  "variables": {},
  "output_messages_to_logs": false
}
```

### 5.3 后端消息处理

**文件**: [websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py#L162-L745)

后端 `WebSocketServer` 类继承自 `tornado.websocket.WebSocketHandler`：

```
on_message(raw_message)
  ├── 解析 JSON，认证校验 (api_key + token)
  ├── 加载 Pipeline 模型
  ├── 权限检查
  └── 分支处理：
      ├── output → 直接转发消息
      ├── cancel_pipeline → 取消管道执行
      ├── check_if_pipeline_running → 检查状态
      ├── execute_pipeline → 执行整个管道
      └── 默认（单 Block 执行）→ __execute_block()
```

**单 Block 执行流程** (`__execute_block`):
1. 获取 Block 模型对象
2. 切换/初始化 Kernel 客户端 (`init_kernel_client`)
3. 根据 Block 类型生成执行代码（`add_execution_code` 注入上下文）
4. 调用 `client.execute(code)` 发送到 Jupyter Kernel
5. 记录 `msg_id` 到 `running_executions_mapping`
6. 如有需要，执行下游 Chart/Widget Block

### 5.4 执行结果回传（后端 → 前端）

后端通过两条路径回传消息：

**路径 1: WebSocket 直接推送**

`WebSocketServer.send_message(message)` → 遍历所有连接客户端 → `client.write_message(json)`

消息格式：
```json
{
  "msg_id": "uuid",
  "type": "text/plain" | "text/html" | "table" | "image/png",
  "data": ["..."] | "...",
  "error": ["..."],
  "execution_state": "busy" | "idle",
  "execution_metadata": {...},
  "msg_type": "stream_pipeline" | null,
  "uuid": "block_uuid",
  "block_type": "transformer",
  "pipeline_uuid": "my_pipeline"
}
```

**路径 2: Subscriber 轮询**

**文件**: [subscriber.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/subscriber.py#L1-L24)

```python
def get_messages(callback=None):
    while True:
        client = get_active_kernel_client()
        message = client.get_iopub_msg(timeout=1)
        if message.get('content'):
            if callback:
                callback(message)  # 回调就是 WebSocketServer.send_message
```

后端独立线程轮询 Jupyter Kernel 的 IOPub 通道，将解析后的消息通过 WebSocket 推送到前端。

### 5.5 前端消息处理

**文件**: [edit.tsx L2431-L2492](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx#L2431-L2492)

```
onMessage(lastMessage)
  ├── JSON.parse() 解析消息
  ├── 提取 uuid, pipeline_uuid, execution_state, msg_type
  ├── msg_type === 'stream_pipeline':
  │   └── 追加到 pipelineMessages，检查是否 IDLE（完成）
  └── 普通 Block 消息:
      ├── 追加到 messages[uuid]（按 Block UUID 分组存储）
      ├── execution_state === 'BUSY' → 加入 runningBlocks
      └── execution_state === 'IDLE' → 移出 runningBlocks
```

消息最终通过 `CodeOutput` 组件渲染展示。

---

## 6. 周边模块配合

### 6.1 REST API 层

**文件**: [api/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/api/index.ts#L1-L521)

采用动态资源路由生成模式，基于 `RESOURCES_PAIRS_ARRAY` 自动生成所有 CRUD 接口：

| 资源 | 说明 | Editor 相关用途 |
|------|------|----------------|
| `pipelines` | 管道管理 | 获取管道详情、保存内容 |
| `blocks` | Block 管理 | 增删改 Block、获取分析结果 |
| `kernels` | Kernel 管理 | 重启/中断内核 |
| `execution_states` | 执行状态 | Spark 任务进度轮询 |
| `autocomplete_items` | 自动补全 | 获取变量、列名等补全项 |
| `block_outputs` | Block 输出 | 获取采样数据 |

API 使用 `react-query` (SWR) 进行数据缓存和自动刷新。

### 6.2 Kernel Context

**文件**: [context/Kernel.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/context/Kernel.tsx#L1-L21)

```typescript
type KernelContextType = {
  fetchKernels: () => void;      // 刷新内核列表
  interruptKernel: () => void;   // 中断执行
  kernel: KernelType;            // 当前内核信息
  restartKernel: () => void;     // 重启内核
};
```

通过 REST API (`api.kernels.useUpdate`) 控制内核状态，执行 `restart` / `interrupt` 动作。

### 6.3 内核管理（后端）

**文件**: [server/kernels.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/kernels.py) + [server/active_kernel.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/active_kernel.py)

- 支持多内核类型：`default_python3`、`pyspark` 等
- `switch_active_kernel(kernel_name)` 切换活动内核
- `get_active_kernel_client()` 获取 Jupyter KernelClient（用于 execute）

### 6.4 Event Stream（SSE）

**文件**: [events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/events/stream.py#L1-L63)

作为 WebSocket 的补充机制，使用 Server-Sent Events (SSE) 通过 HTTP 长连接推送事件流：

```
GET /event-streams/{uuid}
  Content-Type: text/event-stream
  每 100ms 检查队列，有新消息则推送 data: {...}
```

### 6.5 PipelineDetail 组件

**文件**: [PipelineDetail/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/PipelineDetail/index.tsx)

管道详情页核心渲染组件，负责：
- 遍历 blocks 数组，渲染每个 `CodeBlock`
- 管理拖拽（DnD）排序
- 处理 Block 选择、展开/收起
- 集成 ColumnScroller（分屏滚动同步）

### 6.6 CodeBlock 组件

**文件**: [CodeBlock/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/CodeBlock/index.tsx)

单个代码块组件，内部组合：
- `CodeEditor` - 代码编辑区
- `CodeOutput` - 输出展示区
- `CommandButtons` - 运行/保存/删除按钮
- `BlockExtras` - 额外配置（依赖、变量等）
- `SparkProgress` / `SparkJobs` 等 Spark 相关 UI

执行链路：`快捷键` → `executeCode()` → `runBlockAndTrack()` → `runBlock(props.runBlock)` → WebSocket

---

## 7. 关键数据流时序

### 7.1 代码执行完整时序

```
用户         CodeEditor      CodeBlock       edit.tsx       WebSocket      后端Kernel
 │               │               │               │              │             │
 │  Ctrl+Enter   │               │               │              │             │
 │──────────────▶│               │               │              │             │
 │               │ executeCode() │               │              │             │
 │               │──────────────▶│               │              │             │
 │               │               │ runBlockAndTrack()           │             │
 │               │               │─────────────────────────────▶│             │
 │               │               │                               │ sendMessage │
 │               │               │                               │────────────▶│
 │               │               │                               │             │ client.execute()
 │               │               │                               │             │────────────▶
 │               │               │                               │             │             │
 │               │               │                        ◀──────│             │ IOPub msg
 │               │               │                               │             │
 │               │               │  setMessages(append)  ◀──────│             │
 │               │               │                               │             │
 │               │  CodeOutput   │                               │             │
 │               │◀──────────────│                               │             │
 │◀──────────────│    render     │                               │             │
 │   显示输出     │               │                               │             │
```

### 7.2 管道保存时序

```
用户      CodeEditor    CodeBlock    edit.tsx       REST API      后端
  │ save (Ctrl+S) │          │            │            │           │
  │──────────────▶│          │            │            │           │
  │               │ onSave() │            │            │           │
  │               │─────────▶│            │            │           │
  │               │          │ onChangeCodeBlock()     │           │
  │               │          │───────────────────────▶│           │
  │               │          │  (contentByBlockUUID 更新)         │
  │               │          │            │            │           │
  │               │          │ savePipelineContent()  │           │
  │               │          │───────────────────────▶│           │
  │               │          │            │ PUT /pipelines/:uuid │
  │               │          │            │──────────────────────▶│
  │               │          │            │            │           │
  │               │          │            │ ◀──────────│           │
  │               │          │  fetchPipeline()        │           │
  │               │          │───────────────────────▶│           │
```

---

## 8. 状态管理策略

| 状态类型 | 存储位置 | 说明 |
|---------|---------|------|
| Block 代码内容 | `useRef(contentByBlockUUID)` | 非响应式，避免频繁重渲染；保存时收集 |
| Block 输出消息 | `useState(messages: {[uuid]: KernelOutputType[]})` | 按 UUID 分组，WebSocket 推送时追加 |
| 正在运行的 Block | `useState(runningBlocks: BlockType[])` | BUSY 时加入，IDLE 时移出 |
| 编辑器选中状态 | `useState(selectedBlock, textareaFocused)` | 控制聚焦和全局快捷键 |
| 用户偏好设置 | `localStorage` | 分屏开关、隐藏块、自动保存等 |
| API 数据缓存 | `react-query (SWR)` | pipelines、blocks、kernels 等 |

---

## 9. 通信协议关键点

### 9.1 认证

所有 WebSocket 消息必须携带：
- `api_key`: OAuth2 应用 Client ID (`OAUTH2_APPLICATION_CLIENT_ID`)
- `token`: 解码后的 JWT access token

后端通过 `authenticate_client_and_token()` 验证。

### 9.2 重连机制

```typescript
reconnectAttempts: 10,         // 最多重试 10 次
reconnectInterval: 3000,       // 每次间隔 3 秒
shouldReconnect: () => true    // 所有关闭事件都尝试重连
```

### 9.3 敏感数据过滤

后端 `WebSocketServer.send_message()` 在推送前：
- 过滤掉环境变量值（`HIDE_ENV_VAR_VALUES` 开启时）
- 过滤 Jupyter Widget 相关的无法渲染消息
- 美化错误堆栈（去除 Mage 内部调用帧）

---

## 10. 关键文件索引

| 模块 | 文件路径 |
|------|---------|
| **前端入口** | [pages/_app.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/_app.tsx) |
| **Editor 主页面** | [pages/pipelines/[pipeline]/edit.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/pages/pipelines/%5Bpipeline%5D/edit.tsx) |
| **V1 CodeEditor** | [components/CodeEditor/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/CodeEditor/index.tsx) |
| **V2 IDE Manager** | [components/v2/IDE/Manager.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/v2/IDE/Manager.tsx) |
| **CodeBlock** | [components/CodeBlock/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/CodeBlock/index.tsx) |
| **PipelineDetail** | [components/PipelineDetail/index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/components/PipelineDetail/index.tsx) |
| **API 层** | [api/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/api/index.ts) |
| **WS URL 构造** | [api/utils/url.ts](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/api/utils/url.ts) |
| **Kernel Context** | [context/Kernel.tsx](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/frontend/context/Kernel.tsx) |
| **后端 WS Server** | [server/websocket_server.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/websocket_server.py) |
| **后端 Subscriber** | [server/subscriber.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/subscriber.py) |
| **后端 Event Stream** | [server/events/stream.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/events/stream.py) |
| **后端 Server 入口** | [server/server.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/server.py) |
| **后端 Kernels** | [server/kernels.py](file:///d:/fz/0601/solo-dogfeeding/code/318-mage-ai/mage_ai/server/kernels.py) |

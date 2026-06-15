# Mage AI API 服务路由分析

## 一、核心路由注册机制

### 1.1 路由入口

服务启动入口在 `mage_ai/server/server.py`，由 `make_app()` 函数完成所有路由注册。路由按 Tornado 原生的 `(pattern, Handler, kwargs)` 三元组列表组织，**按注册顺序匹配，先命中先处理**。

### 1.2 路由分层结构

```
routes_base (健康检查专用)
  └── /api/status(es) ── 健康检查，可绕过 OAuth

routes_full = routes_base + [...]
  ├── 页面路由（SPA 前端入口）
  │     ├── /, /files, /overview, /pipelines, ...
  │     └── 全部指向 MainPageHandler，渲染 index.html 由前端接管
  │
  ├── 静态资源路由
  │     ├── /_next/static/*, /fonts/*, /images/*, /monaco-editor/*
  │     └── /favicon.ico
  │
  ├── WebSocket 路由
  │     ├── /websocket/ ── WebSocketServer（内核输出）
  │     └── /websocket/terminal ── TerminalWebsocketServer
  │
  ├── 专用 API 路由（优先匹配，避免被通用路由截断）
  │     ├── /api/events, /api/event_matchers/*
  │     ├── /api/pipelines/{uuid}/blocks/{uuid}/analyses
  │     ├── /api/pipeline_schedules/*/pipeline_runs/{token}（向后兼容）
  │     ├── /api/pipeline_schedules/*/api_trigger
  │     ├── /api/runs, /api/runs/{token}
  │     ├── /api/pipelines/{uuid}/block_outputs/{uuid}/downloads
  │     ├── /api/downloads/{token}
  │     ├── /api/file_contents/{pk}（含路径，需特殊正则）
  │     ├── /api/pipelines/{pk}/blocks/{child_pk}（嵌套资源显式注册）
  │     ├── /api/files/{pk}/file_versions
  │     ├── /api/page_block_layouts/{pk}, /api/block_outputs/{pk}
  │     └── /api/git_custom_branches/{pk}
  │
  └── 通用 RESTful API 路由（兜底匹配）
        ├── /api/{resource}/{pk}/{child}/{child_pk} → ApiChildDetailHandler
        ├── /api/{resource}/{pk}/{child}           → ApiChildListHandler
        ├── /api/{resource}/{pk}                   → ApiResourceDetailHandler
        └── /api/{resource}                        → ApiResourceListHandler
```

### 1.3 关键路由约束

- **路径穿越防护**：`PATH_TRAVERSAL_PATTERN = r"^(?!.*(\.\.\/|\.\.\\)).*$"`，在 `BaseApiHandler.prepare()` 中对 URL path 和所有 query parameter 值做检查，先 URL decode 再检测
- **Base Path 替换**：若设置 `ROUTES_BASE_PATH`，所有路由前缀自动替换，前端静态资源同步替换占位符
- **Status-only 模式**：`status_only=True` 时仅注册健康检查路由，用于调度器节点

---

## 二、Handler 继承与预处理链

### 2.1 继承体系

```
tornado.web.RequestHandler
  │
  ├── BaseHandler (mage_ai/server/api/base.py)
  │     ├── set_default_headers() ── CORS 跨域处理
  │     ├── write() ── JSON 序列化统一封装（simplejson + encode_complex）
  │     ├── write_error() ── 500 错误上报 UsageStatisticLogger
  │     ├── limit() ── 分页辅助（_limit, _offset 参数）
  │     └── get_payload() ── Body 解析，按 model_class 提取单根键
  │
  └── BaseApiHandler (BaseHandler + OAuthMiddleware)
        └── prepare() 生命周期钩子链
              ├── 更新用户活动时间戳（ActivityTracker）
              ├── URL path 解码 + 路径穿越检测 → 400 拦截
              ├── 所有 query parameter 值解码 + 路径穿越检测 → 400 拦截
              └── super().prepare() → OAuthMiddleware.prepare()
```

### 2.2 OAuth 认证中间件（OAuthMiddleware.prepare()）

**认证信息提取顺序**：
1. **API Key 提取**：
   - Header: `X-API-KEY`
   - POST Body: `api_key` 字段
   - Query: `?api_key=xxx`
2. **OAuth Token 提取**：
   - Header: `OAUTH-TOKEN`
   - Header: `Authorization: Bearer xxx`
   - Cookie: `oauth_token`（JWT 解码）

**认证校验流程**：
- API Key → 匹配 `Oauth2Application.client_id`（必须等于预置的 `OAUTH2_APPLICATION_CLIENT_ID`）
- Token → `authenticate_client_and_token()` 校验有效性和过期
- 通过则注入 `request.current_user`、`request.oauth_client`、`request.oauth_token`
- 失败则设置 `request.error`（不抛出异常，由下游读取）

**Scope 判定**（BasePolicy.current_scope()）：
- 已登录 + 启用认证 → `CLIENT_PRIVATE`
- 未登录 + 禁用编辑访问 → 也视为 `CLIENT_PRIVATE`（阻止匿名编辑）
- 其他 → `CLIENT_PUBLIC`

---

## 三、请求处理完整调用链（核准顺序）

### 3.1 通用入口流程

以 `GET /api/pipelines/test-pipeline-123` 为例：

```
1. Tornado 路由匹配
   /api/(?P<resource>\w+)/(?P<pk>[\w\-\%2f\.]+)
   → ApiResourceDetailHandler
   → resource="pipelines", pk="test-pipeline-123"

2. prepare() 预处理链
   ├── BaseApiHandler.prepare()
   │     ├── 路径安全检测
   │     └── 查询参数安全检测
   └── OAuthMiddleware.prepare()
         ├── 提取并校验 API Key
         ├── 提取并校验 OAuth Token
         └── 注入 request.current_user / oauth_token / error

3. Handler 方法调用
   ApiResourceDetailHandler.get(resource, pk)
   → execute_operation(handler, resource, pk=pk)

4. views.py 调度层（mage_ai/api/views.py）
   ├── __determine_action() 方法映射：
   │     GET + pk → DETAIL
   │     GET (无 pk) → LIST
   │     POST → CREATE
   │     PUT → UPDATE
   │     DELETE → DELETE
   │
   ├── __query()：提取非 _ 前缀的 query 参数 → 业务过滤条件
   ├── __meta()：提取 _ 前缀的 query 参数 → 分页/排序/格式控制
   ├── __payload()：解析 Body（支持 JSON 和 multipart 的 json_root_body 字段）
   │
   ├── 构建 BaseOperation 对象
   │     ├── action="detail"
   │     ├── resource="pipelines"（后续 singularize → "pipeline"）
   │     ├── pk="test-pipeline-123"
   │     ├── user, oauth_token, meta, query, payload
   │     └── resource_parent / resource_parent_id（嵌套资源时）
   │
   └── base_operation.execute()

5. BaseOperation.execute()（核心编排，mage_ai/api/operations/base.py）
   ├── __executed_result() → 业务结果获取（见 3.2 / 3.3 分路径）
   ├── __present_results() → Presenter 展示转换（第1次）
   ├── __run_hooks_after(SUCCESS) → 成功分支 AFTER Hooks
   ├── 遍历结果 → 字段级二次 READ 授权 + Parser 过滤（第2次授权）
   └── 组装响应 + 异常处理

6. 响应写回
   ├── simplejson.dumps + encode_complex 处理 datetime/Decimal
   └── 记录 API 延迟指标
```

### 3.2 路径 A：DELETE / DETAIL / UPDATE 执行顺序

**核心特征：先加载业务对象，再做授权**（因为授权条件可能依赖对象本身属性）

```
__delete_show_or_update() 执行顺序：
  │
  1. __updated_options() → 加载父模型（嵌套资源时）
  2. process_member(pk, user, ...) → 加载业务对象 ✅ 先加载
  3. __run_hooks_before() → BEFORE Hooks（可修改 payload）
  4. policy = Policy(resource, user, ...) → 创建策略对象
  5. policy.authorize_action(ACTION) → 动作级授权 ✅ 后授权
  6. Parser 初始化 + 字段/查询授权：
     ├── DETAIL: parse_query_and_authorize() + authorize_query()
     ├── UPDATE: parse_write_attributes_and_authorize(WRITE)
     │          + parse_query_and_authorize()
     └── DELETE: （无额外授权）
  7. 执行具体操作：
     ├── DELETE: resource.process_delete()
     ├── UPDATE: resource.process_update(payload)
     └── DETAIL: 已在第 2 步加载完成
  8. return resource 对象
```

### 3.3 路径 B：CREATE / LIST 执行顺序

**核心特征：先授权，再执行业务操作**（无需预加载对象）

```
__create_or_index() 执行顺序：
  │
  1. __updated_options() → 加载父模型
  2. __run_hooks_before() → BEFORE Hooks（可修改 payload/query/meta）
  3. policy = Policy(None, user, ...) → 创建策略对象（resource=None）
  4. policy.authorize_action(ACTION) → 动作级授权 ✅ 先授权
  5. Parser 初始化 + 字段/查询授权：
     ├── CREATE: parse_write_attributes_and_authorize(WRITE)
     └── LIST: parse_query_and_authorize() + authorize_query()
  6. 执行具体操作：
     ├── CREATE: Resource.process_create(payload, user, ...) ✅ 后创建
     └── LIST: Resource.process_collection(query, meta, user, ...) ✅ 后查询
  7. return result 对象
```

---

## 四、完整执行时序图

### 4.1 GET /api/pipelines/{pk} 时序

```
客户端 → Tornado → ApiResourceDetailHandler
                     │
                     ├─ prepare()
                     │   ├─ 路径安全检测 ✓
                     │   └─ OAuth 认证 ✓
                     │
                     ├─ get(resource, pk)
                     │   └─ execute_operation()
                     │        ├─ __determine_action() → DETAIL
                     │        ├─ __query(), __meta(), __payload()
                     │        └─ BaseOperation.execute()
                     │             │
                     │             ├─ __delete_show_or_update()
                     │             │   ├─ __updated_options() → 加载父模型
                     │             │   ├─ PipelineResource.process_member(pk)  ← 1. 加载对象
                     │             │   ├─ __run_hooks_before()                ← 2. BEFORE Hooks
                     │             │   ├─ PipelinePolicy.authorize_action(DETAIL)  ← 3. 动作授权
                     │             │   └─ Parser.parse_query_and_authorize()  ← 4. 查询授权
                     │             │
                     │             ├─ PipelinePresenter.present_resource()    ← 5. 展示转换
                     │             ├─ __run_hooks_after(SUCCESS)              ← 6. AFTER Hooks
                     │             │
                     │             └─ 遍历结果，二次授权：
                     │                  policy.authorize_attributes(READ, fields)  ← 7. 字段授权
                     │                  Parser.parse_read_attributes_and_authorize() ← 8. Parser 过滤
                     │
                     └─ handler.write(json_response)
```

### 4.2 POST /api/pipelines 时序

```
客户端 → Tornado → ApiResourceListHandler
                     │
                     ├─ prepare()
                     │   ├─ 路径安全检测 ✓
                     │   └─ OAuth 认证 ✓
                     │
                     ├─ post(resource)
                     │   └─ execute_operation()
                     │        ├─ __determine_action() → CREATE
                     │        └─ BaseOperation.execute()
                     │             │
                     │             ├─ __create_or_index()
                     │             │   ├─ __updated_options() → 加载父模型
                     │             │   ├─ __run_hooks_before()                ← 1. BEFORE Hooks
                     │             │   ├─ PipelinePolicy.authorize_action(CREATE)  ← 2. 动作授权
                     │             │   ├─ Parser.parse_write_attributes_and_authorize(WRITE)  ← 3. 字段授权
                     │             │   └─ PipelineResource.process_create(payload)  ← 4. 执行业务
                     │             │
                     │             ├─ PipelinePresenter.present_resource()    ← 5. 展示转换
                     │             ├─ __run_hooks_after(SUCCESS)              ← 6. AFTER Hooks
                     │             │
                     │             └─ 二次授权：authorize_attributes(READ)   ← 7. 字段级 READ 授权
                     │
                     └─ handler.write(json_response)
```

---

## 五、各协作组件顺序与职责

### 5.1 执行顺序汇总表

| 阶段 | 组件 | 职责 | 调用时机 |
|------|------|------|---------|
| 1 | **Tornado Router** | URL 模式匹配 → Handler | 最外层 |
| 2 | **BaseApiHandler.prepare()** | 路径安全检测 | prepare() 第一环 |
| 3 | **OAuthMiddleware.prepare()** | 认证（API Key + Token + Cookie） | prepare() 第二环 |
| 4 | **execute_operation()** | 请求解析、确定 action、构建 BaseOperation | Handler 方法第一行 |
| 5 | **GlobalHooks BEFORE** | 修改 payload / query / meta | 业务操作前 |
| 6 | **Policy.authorize_action()** | 动作级授权（能否执行该操作） | 业务操作前 |
| 7 | **Parser.parse_*_and_authorize()** | 输入解析 + 字段/查询授权 | 业务操作前 |
| 8 | **Resource.process_*()** | 实际业务逻辑（增删改查） | 授权通过后 |
| 9 | **Presenter.present_resource()** | 数据模型 → API 展示格式转换 | 业务操作后 |
| 10 | **GlobalHooks AFTER** | 修改结果、metadata、注入 error | 展示转换后 |
| 11 | **Policy.authorize_attributes(READ)** | 字段级二次授权（读过滤） | AFTER Hooks 后 |
| 12 | **Parser.parse_read_attributes_and_authorize()** | 输出过滤 + 字段级 READ 授权 | 二次授权时 |

### 5.2 核心协作类约定（动态加载）

`resource="pipelines"` 经过 `singularize → classify → "Pipeline"`，然后动态 import：

| 类名 | 必须 | 职责 | 路径 |
|------|------|------|------|
| `PipelineResource` | ✅ | 业务逻辑（process_create/member/collection） | `mage_ai/api/resources/PipelineResource.py` |
| `PipelinePolicy` | ✅ | 授权规则声明与校验 | `mage_ai/api/policies/PipelinePolicy.py` |
| `PipelinePresenter` | ✅ | 展示格式转换 | `mage_ai/api/presenters/PipelinePresenter.py` |
| `PipelineParser` | ❌ | 输入输出解析转换 | `mage_ai/api/parsers/PipelineParser.py` |
| `PipelineMonitor` | ❌ | 错误包装（默认 BaseMonitor） | `mage_ai/api/monitors/PipelineMonitor.py` |

**新增资源零配置**：按命名约定创建上述文件，路由自动生效。

---

## 六、重要约束与设计要点

### 6.1 权限三重漏斗模型

Policy 授权采用从粗到细的三层校验：

```
1. authorize_action() ── 动作级
   能否对该资源执行 CREATE/LIST/DETAIL/UPDATE/DELETE？
   规则来源: Policy.allow_actions()

   ├─ 2. authorize_query() ── 查询参数级
   │      能否按该条件过滤？(如 ?status=active&type=python)
   │      规则来源: Policy.allow_query()

   └─ 3. authorize_attributes() ── 字段级
          能否读/写该字段？(如读取 password_hash、修改 user.role)
          规则来源: Policy.allow_read() / allow_write()
```

每条规则的三要素：
- **Scope**：`CLIENT_PUBLIC` / `CLIENT_PRIVATE` / `CLIENT_ALL`
- **Condition**：回调函数，如 `lambda p: p.has_at_least_editor_role()`
- **On Action**：绑定到特定操作（如仅 `UPDATE` 允许修改某字段）

### 6.2 请求参数拆分约束

在 `views.py` 中，请求数据被拆分为四个独立通道，系统级与业务级彻底分离：

| 数据类型 | 提取规则 | 用途 |
|---------|---------|------|
| `meta` | Query 参数中 `_` 开头的键（`_limit`, `_format`, `_order_by[]`） | 分页、排序、展示格式 |
| `query` | Query 参数中非 `_` 开头的键（`?status=active`） | 业务过滤条件 |
| `payload` | Body 内容（JSON 或 multipart 的 `json_root_body`） | 创建/更新数据 |
| `options` | Query arguments 原始值 | 内部方法透传 |

### 6.3 全局 Hooks 约束

- **BEFORE Hooks**：可修改 `payload` / `meta` / `query`
- **AFTER Hooks**：成功分支可修改 `resource(s)` / `metadata` / 注入 `error`；失败分支可修改 `error` / `metadata`
- **终止策略**：`HookStrategy.RAISE` → 抛出异常中断流程
- **启用条件**：`FeatureUUID.GLOBAL_HOOKS` 特性开关

### 6.4 关键注意事项

1. **路由顺序不可乱**：含 `%2f`、`.` 的路径类资源必须放在通用路由之前，否则正则匹配会截断路径
2. **类命名严格约定**：资源名复数 → 单数 → 驼峰，否则 `ModuleNotFoundError`
3. **认证错误不抛异常**：中间件仅设置 `request.error`，由 `execute_operation` 读取渲染
4. **两条路径顺序不同**：DELETE/DETAIL/UPDATE 先加载后授权，CREATE/LIST 先授权后操作
5. **授权做两次**：第一次在业务操作前（WRITE/QUERY 权限），第二次在展示后（READ 字段过滤）
6. **异步同步混用**：`BaseOperation.execute()` 是 async，但 Resource 内部部分方法仍用同步 I/O，注意阻塞

---

## 七、核心文件索引

| 层级 | 相对路径 | 关键职责 |
|------|---------|---------|
| 服务启动 | `mage_ai/server/server.py` | 路由注册、服务器启动、缓存初始化 |
| Handler 基类 | `mage_ai/server/api/base.py` | BaseHandler / BaseApiHandler、路径安全 |
| 通用 Handler | `mage_ai/server/api/v1.py` | 四个通用 REST Handler 类 |
| 视图调度 | `mage_ai/api/views.py` | `execute_operation()` 请求解析与调度 |
| 认证中间件 | `mage_ai/api/middleware.py` | OAuthMiddleware 三重认证 |
| 业务编排 | `mage_ai/api/operations/base.py` | BaseOperation 全流程编排 |
| 资源基类 | `mage_ai/api/resources/BaseResource.py` | Resource 层接口定义 |
| 策略基类 | `mage_ai/api/policies/BasePolicy.py` | 声明式权限规则与授权校验 |
| 操作常量 | `mage_ai/api/operations/constants.py` | CREATE/LIST/DETAIL/UPDATE/DELETE |
| 安全常量 | `mage_ai/server/api/constants.py` | 路径穿越正则、认证头常量 |

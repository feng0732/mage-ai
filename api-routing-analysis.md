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

### 3.1 通用入口流程（BaseOperation.execute() 外层）

以 `GET /api/pipelines/test-pipeline-123` 为例，所有 action 类型共享的外层流程：

```
1. Tornado 路由匹配
   /api/(?P<resource>\w+)/(?P<pk>[\w\-\%2f\.]+)
   → ApiResourceDetailHandler
   → resource="pipelines", pk="test-pipeline-123"

2. prepare() 预处理链（Tornado 生命周期）
   ├── BaseApiHandler.prepare()
   │     ├── 更新用户活动时间戳（ActivityTracker）
   │     ├── URL path 解码 + 路径穿越检测（400 拦截）
   │     └── 查询参数解码 + 路径穿越检测（400 拦截）
   └── OAuthMiddleware.prepare()
         ├── 提取并校验 API Key（Header/Body/Query）
         ├── 提取并校验 OAuth Token（Header/Cookie）
         └── 注入 request.current_user / oauth_client / oauth_token / error

3. Handler 方法调用
   ApiResourceDetailHandler.get(resource, pk)
   └── 第一行即调用 execute_operation(handler, resource, pk=pk)

4. views.py 调度层（mage_ai/api/views.py: execute_operation()）
   ├── __determine_action() 方法映射：
   │     ├── DELETE                       → DELETE
   │     ├── GET  + pk + child + child_pk → DETAIL
   │     ├── GET  + pk + child            → LIST
   │     ├── GET  + pk                    → DETAIL
   │     ├── GET  (无 pk)                 → LIST
   │     ├── POST                         → CREATE
   │     └── PUT                          → UPDATE
   │
   ├── __query()：提取非 _ 前缀的 query 参数 → 业务过滤条件
   ├── __meta()：提取 _ 前缀的 query 参数 → 分页/排序/格式控制
   ├── __payload()：解析 Body（JSON 或 multipart 的 json_root_body 字段）
   │
   ├── 构建 BaseOperation 对象：
   │     ├── action, resource, pk, child, child_pk
   │     ├── user, oauth_client, oauth_token, error（来自 request）
   │     ├── meta, query, payload, options, files, headers
   │     └── resource_parent / resource_parent_id（嵌套资源时）
   │
   └── 进入核心编排：await base_operation.execute()

5. BaseOperation.execute() 主流程（mage_ai/api/operations/base.py: L77-L262）
   ├── ① __executed_result() → 走两条子路径之一（见 3.2 / 3.3）
   ├── ② __present_results(result) → Presenter 展示转换（第 1 次）
   ├── ③ 从 ResultSet 提取 metadata
   ├── ④ __run_hooks_after(SUCCESS) → AFTER Hooks（可改结果/metadata/注入 error）
   ├── ⑤ 遍历每条结果 → 字段级二次 READ 授权 + Parser 过滤（第 2 次授权）
   ├── ⑥ 组装响应 { "pipelines": [...], "metadata": {...}, "debug": {...} }
   └── 异常分支：ApiError → Monitor 包装 → __run_hooks_after(FAILURE)

6. 响应写回（views.py 尾部）
   ├── simplejson.dumps + encode_complex（处理 datetime/Decimal/NaN 等）
   └── Prometheus metrics：api.time / sql.time 打点
```

> **两条子路径的分叉点**：在第 5 步的 `__executed_result()` 内部，依据 action 进入不同函数。

### 3.2 路径 A：DELETE / DETAIL / UPDATE 执行顺序（`__delete_show_or_update` L582-L712）

**核心特征：先加载 Resource，再跑 BEFORE Hooks，再做授权**
> 原因：授权条件可能依赖已加载对象的属性（如判断"是否是自己创建的资源"）；BEFORE Hooks 也需要拿到对象才能判断。

```
__delete_show_or_update() 执行顺序（逐行核准）：

  L583  1. __updated_options()
          ├── 合并 context/meta/query/payload/options
          ├── 注入 oauth_client / oauth_token / api_operation_action
          └── __parent_model() → 嵌套资源时预加载父对象

  L585  2. Resource.process_member(pk, user, **updated_options)  ← ✅ 先加载业务对象
          （Resource 类通过动态 import：resource 名 → singularize → classify → XxxResource）

  L594  3. __run_hooks_before()  ← BEFORE Hooks
          ├── UPDATE: 先调 __payload_for_resource() 提取 payload，再跑 Hooks
          ├── DETAIL/DELETE: payload=None，以第 2 步加载的 res 为 operation_resource
          └── 可修改 payload / meta / query

  L599  4. Policy 实例化 + 动作级授权
          ├── Policy(resource=res, user, **updated_options)  ← 绑定了已加载对象
          └── policy.authorize_action(self.action)             ← ✅ 后授权（动作级）
                ├── scope 校验（CLIENT_PUBLIC / CLIENT_PRIVATE）
                └── condition 回调校验（角色 / 自定义）

  L602  5. Parser 实例化
          parser = ParserClass(resource=res, user=user, policy=policy, **updated_options)
          （Parser 可选，不存在则跳过所有 Parser 步骤）

  L612  6. Parser 解析 + 字段/查询授权（按 action 分支）：
          ┌─ DELETE（L612-L613）:
          │    无 Parser 调用，直接执行 res.process_delete()
          │
          ├─ DETAIL（L614-L661）:
          │    Parser.parse_query_and_authorize(self.query, build_authorize_query)
          │    └── 内部调用 authorize_query() → 查询参数级授权
          │    特殊：如 parser_found + error，会用修改后的 query 重新 process_member
          │
          └─ UPDATE（L663-L710）:
               ① Parser.parse_write_attributes_and_authorize(payload, build_auth_attrs)
                 └── 内部调用 authorize_attributes(WRITE) → 字段级写授权
               ② Parser.parse_query_and_authorize(self.query, build_authorize_query)
                 └── 内部调用 authorize_query() → 查询参数级授权
               ③ 执行 res.process_update(payload, query=...)

  L712  7. return res  ← 返回 Resource 对象（可能已被 DELETE/UPDATE 修改状态）
```

### 3.3 路径 B：CREATE / LIST 执行顺序（`__create_or_index` L485-L580）

**核心特征：先跑 BEFORE Hooks，再做授权，最后调用 Resource**
> 原因：无需预加载对象；CREATE 的 payload 经 Hooks 改造后再做权限校验更合理。

```
__create_or_index() 执行顺序（逐行核准）：

  L486  1. __updated_options()
          与路径 A 完全相同：合并上下文 + 加载父模型

  L493  2. __run_hooks_before()  ← ✅ 先跑 BEFORE Hooks
          ├── CREATE: 先 __payload_for_resource() 提取 payload，再跑 Hooks
          └── LIST: payload=None，operation_resource=None
          Hooks 可修改 payload / meta / query

  L500  3. Policy 实例化 + 动作级授权
          ├── Policy(resource=None, user, **updated_options)  ← resource 为 None！
          └── policy.authorize_action(self.action)             ← ✅ 先授权（动作级）
                （与路径 A 同样的 scope + condition 校验）

  L503  4. Parser 实例化
          parser = ParserClass(resource=None, user=user, policy=policy, **updated_options)
          resource=None，与路径 A 不同！

  L513  5. Parser 解析 + 字段/查询授权（按 action 分支）：
          ┌─ CREATE（L513-L541）:
          │    Parser.parse_write_attributes_and_authorize(payload, build_auth_attrs)
          │    └── 内部调用 authorize_attributes(WRITE, payload.keys()) → 字段级写授权
          │    无 Parser 时直接调 _build_authorize_attributes(payload)
          │
          └─ LIST（L542-L580）:
               Parser.parse_query_and_authorize(self.query, build_authorize_query)
               └── 内部调用 authorize_query() → 查询参数级授权
               无 Parser 时直接调 _build_authorize_query(self.query)

  L534  6. Resource 业务调用  ← ✅ 最后才调用 Resource
          ┌─ CREATE:
          │   XxxResource.process_create(payload, user, result_set_from_external, **options)
          │
          └─ LIST:
               XxxResource.process_collection(query, meta, user, result_set_from_external, **options)

  L541/  7. return result
  L580     （CREATE 返回单个 Resource；LIST 返回列表/ResultSet）
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

### 5.1 公共流程顺序（所有 action 一致，BaseOperation.execute() 外层 L77-L262）

两条路径（DELETE/DETAIL/UPDATE 与 CREATE/LIST）只在 `__executed_result()` 内部有差异。
进入 `__executed_result()` 之前、以及它返回之后的所有步骤，所有 action 完全一致：

| 阶段 | 组件 | 职责 | 代码位置 |
|------|------|------|---------|
| 1 | **Tornado Router** | URL 正则匹配 → Handler + 命名分组 kwargs | `server.py: make_app()` |
| 2 | **BaseApiHandler.prepare()** | 更新活动时间戳 + 路径穿越检测（path + 所有 query 值） | `server/api/base.py` |
| 3 | **OAuthMiddleware.prepare()** | 提取并校验 API Key / Token，注入 user / oauth_client / oauth_token / error | `api/middleware.py` |
| 4 | **execute_operation()** | `__determine_action()` 映射 action；拆分 meta/query/payload；构造 BaseOperation | `api/views.py: L15-L105` |
| 5 | **`__executed_result()`** | **分叉点** —— 按 action 走两条子路径（差异见 5.2） | `operations/base.py: L479-L483` |
| 6 | **Presenter.present_resource()** | Resource 对象 → dict 格式转换（含嵌套资源） | `presenters/XxxPresenter.py` |
| 7 | **ResultSet.metadata** | 提取分页 / 总数等元数据（仅 LIST 返回 ResultSet 时生效） | L96-L97 |
| 8 | **`__run_hooks_after(SUCCESS)`** | AFTER Hooks，可修改 presented 结果 / metadata / 注入 error | `GlobalHooks` 特性 |
| 9 | **Policy.authorize_attributes(READ)** | 逐条结果遍历，字段级二次 READ 授权（从 presented dict 取 keys） | L147-L163 |
| 10 | **Parser.parse_read_attributes_and_authorize()** | 输出字段再过滤 + 再授权（与第 9 步同一次循环，Parser 存在时调用） | L175-L176 |
| 11 | **组装响应 dict** | `{ "resource": {...}, "metadata": {...}, "debug": {...} }`（含异常分支 Monitor 包装） | L187-L258 |
| 12 | **simplejson.dumps** | encode_complex 处理 datetime / Decimal / NaN → JSON 字符串 | `views.py` 尾部 |

> **注意**：上表中第 5~8 步是线性的，在所有 action 中均按此顺序执行。两条路径的差异 **仅发生在第 5 步内部**（`__executed_result()`）。

---

### 5.2 两条路径差异对照表（第 5 步 `__executed_result()` 内部）

这是之前冲突的根源：**BEFORE Hooks 与 Resource 加载/调用的顺序，在两条路径中正好相反。**

| 步骤 | DELETE / DETAIL / UPDATE（`__delete_show_or_update` L582-L712） | CREATE / LIST（`__create_or_index` L485-L580） |
|------|-----------------------------------------------------------------|-----------------------------------------------|
| **1** | `__updated_options()` 合并上下文 + 加载父模型（共同） | `__updated_options()` 合并上下文 + 加载父模型（共同） |
| **2** | ✅ **Resource.process_member(pk)** → 先加载业务对象 | `__run_hooks_before()` → 先跑 BEFORE Hooks |
| **3** | `__run_hooks_before()` → 后跑 BEFORE Hooks（已拿到对象） | **Policy(resource=None, ...)** → 实例化（resource 为 None） |
| **4** | **Policy(resource=res, ...)** → 绑定已加载对象实例化 | **authorize_action()** → 先做动作级授权 |
| **5** | **authorize_action()** → 后做动作级授权（条件可依赖对象属性） | Parser 实例化（resource=None） |
| **6** | Parser 实例化（resource=res，已绑定对象） | Parser 解析 + 授权（见 5.3 各 action 明细） |
| **7** | Parser 解析 + 授权（见 5.3 各 action 明细） | ✅ **Resource.process_create / process_collection** → 最后才调用 Resource |
| **8** | ✅ **执行 DELETE / UPDATE / DETAIL**（DETAIL 已在第 2 步完成加载） | 返回结果对象 |
| **9** | 返回结果对象（可能已被修改状态） | — |

**顺序差异核心要点**：

- **DELETE/DETAIL/UPDATE**：第 2 步加载 Resource → 第 3 步 BEFORE Hooks → 第 5 步授权 → 第 8 步执行业务
- **CREATE/LIST**：第 2 步 BEFORE Hooks → 第 4 步授权 → 第 7 步调用 Resource
- **Policy 实例化参数不同**：路径 A 传入 `resource=res`（已加载对象），路径 B 传入 `resource=None`
- **Parser 实例化参数不同**：同上，路径 A 的 Parser 可访问已加载对象

---

### 5.3 各 action 解析/授权明细（两条路径内部的 Parser 分支）

同样是 Parser，5 种 action 实际调用的方法和授权层级不同：

| Action | 所属路径 | Parser 调用（业务操作前） | 授权层级 | 最终 Resource 调用 |
|--------|---------|--------------------------|---------|-------------------|
| **DETAIL** | 路径 A | `parse_query_and_authorize(query, build_authorize_query)` | authorize_query（查询级） | 已在第 2 步通过 `process_member()` 加载完成 |
| **DELETE** | 路径 A | ❌ 无 Parser 调用 | 仅 authorize_action（动作级） | `res.process_delete()` |
| **UPDATE** | 路径 A | ① `parse_write_attributes_and_authorize(payload, build_auth_attrs)`  <br> ② `parse_query_and_authorize(query, build_authorize_query)` | authorize_attributes(WRITE)（字段级写） + authorize_query（查询级） | `res.process_update(payload, query=...)` |
| **CREATE** | 路径 B | `parse_write_attributes_and_authorize(payload, build_auth_attrs)` | authorize_attributes(WRITE)（字段级写） | `XxxResource.process_create(payload, ...)` |
| **LIST** | 路径 B | `parse_query_and_authorize(query, build_authorize_query)` | authorize_query（查询级） | `XxxResource.process_collection(query, meta, ...)` |

> **所有 action 的公共授权（业务操作后）**：在 execute() 第 9、10 步还会再做一次
> `authorize_attributes(READ)` + `parse_read_attributes_and_authorize()` 的字段级读过滤。
> 即：写操作会经历 WRITE 授权 + READ 授权两次；读操作会经历 QUERY 授权 + READ 授权两次。

---

### 5.4 授权的完整生命周期（双路径 + 三重漏斗 + 前后双段）

> **重要**：授权层级（动作级 → 查询/字段级 → 字段级读过滤）是一致的，但
> **Resource 加载/调用与 authorize_action() 的先后顺序在两条路径上相反**。

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           业务操作前（第 5 步 __executed_result() 内部）              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│   ┌──────────────────────── 路径 A：DELETE / DETAIL / UPDATE ──────────────────────┐│
│   │                                                                                ││
│   │  ① Resource.process_member(pk)  ← 先加载对象                                    ││
│   │     （已拿到 res，Policy 条件可依赖对象属性）                                     ││
│   │                                                                                ││
│   │  ② __run_hooks_before()  ← BEFORE Hooks                                        ││
│   │                                                                                ││
│   │  ③ policy.authorize_action(ACTION)  ← 漏斗 1：动作级授权（后做）                 ││
│   │     └─ Policy 实例化时传入 resource=res                                          ││
│   │                                                                                ││
│   │  ④ Parser 解析 + 授权（漏斗 2）：                                               ││
│   │     ├─ DELETE: 无 Parser                                                        ││
│   │     ├─ DETAIL: parse_query_and_authorize()                                      ││
│   │     │          → authorize_query()  ← 查询级                                     ││
│   │     └─ UPDATE: ① parse_write_attributes_and_authorize()                         ││
│   │                  → authorize_attributes(WRITE)  ← 字段级写                       ││
│   │               ② parse_query_and_authorize()                                     ││
│   │                  → authorize_query()  ← 查询级                                   ││
│   │                                                                                ││
│   │  ⑤ 执行业务：                                                                   ││
│   │     ├─ DELETE: res.process_delete()                                             ││
│   │     ├─ UPDATE: res.process_update(payload)                                      ││
│   │     └─ DETAIL: （已在第 ① 步加载完成）                                           ││
│   │                                                                                ││
│   └─────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                     │
│   ┌──────────────────────── 路径 B：CREATE / LIST ────────────────────────────────┐ │
│   │                                                                                │ │
│   │  ① __run_hooks_before()  ← BEFORE Hooks（先跑）                                 │ │
│   │                                                                                │ │
│   │  ② policy.authorize_action(ACTION)  ← 漏斗 1：动作级授权（先做）                 │ │
│   │     └─ Policy 实例化时传入 resource=None                                         │ │
│   │                                                                                │ │
│   │  ③ Parser 解析 + 授权（漏斗 2）：                                               │ │
│   │     ├─ CREATE: parse_write_attributes_and_authorize()                           │ │
│   │     │          → authorize_attributes(WRITE)  ← 字段级写                         │ │
│   │     └─ LIST:   parse_query_and_authorize()                                      │ │
│   │                → authorize_query()  ← 查询级                                     │ │
│   │                                                                                │ │
│   │  ④ Resource 业务调用  ← 最后才调用                                               │ │
│   │     ├─ CREATE: XxxResource.process_create(payload)                               │ │
│   │     └─ LIST:   XxxResource.process_collection(query, meta)                       │ │
│   │                                                                                │ │
│   └─────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
│   共同点：漏斗 1（动作级）→ 漏斗 2（查询/字段级写）→ Resource 业务操作                │
│   差异点：漏斗 1 之前，路径 A 先加载 Resource，路径 B 先跑 Hooks                     │
│                                                                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                         业务操作后（execute() 第 9、10 步，所有 action 一致）          │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ① Presenter.present_resource()  → Resource 对象转成 dict（可能含嵌套资源）          │
│                                                                                     │
│  ② __run_hooks_after(SUCCESS)  ← AFTER Hooks                                       │
│     可修改 presented 结果 / metadata / 注入 error                                   │
│                                                                                     │
│  ③ 遍历每条结果，漏斗 3：字段级读授权                                                │
│     ├─ policy.authorize_attributes(READ, presented.keys())                          │
│     │    ← 对 presented dict 的每个 key 检查 READ 权限                               │
│     │                                                                               │
│     └─ Parser（存在时）：                                                            │
│        parse_read_attributes_and_authorize()  → 输出字段再过滤 + 再授权             │
│                                                                                     │
│  ④ 组装最终响应：{ "resource": {...}, "metadata": {...}, "debug": {...} }           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

> 总结一句：**成员资源（DELETE/DETAIL/UPDATE）先加载对象再授权，集合创建（CREATE/LIST）先授权后调用 Resource。**
> 授权的"三层漏斗"概念是一致的——动作级最粗、查询/字段级居中、读过滤最细——但 Resource 调用与第一层授权的先后顺序因路径而异。

---

### 5.5 核心协作类约定（动态加载）

`resource="pipelines"` 字符串的转换链：
```
"pipelines" → singularize → "pipeline" → classify → "Pipeline"
```

然后通过 `importlib.import_module(f'mage_ai.api.resources.PipelineResource')` 动态加载：

| 类名 | 必须 | 职责 | 相对路径 |
|------|------|------|---------|
| `XxxResource` | ✅ | 业务逻辑（process_create / member / collection / delete / update） | `mage_ai/api/resources/XxxResource.py` |
| `XxxPolicy` | ✅ | 权限规则声明（allow_actions / allow_read / allow_write / allow_query） + 授权校验 | `mage_ai/api/policies/XxxPolicy.py` |
| `XxxPresenter` | ✅ | 数据模型 → API 展示格式转换（支持嵌套资源展示） | `mage_ai/api/presenters/XxxPresenter.py` |
| `XxxParser` | ❌ | 输入解析转换、查询/写授权执行、输出过滤（不存在则跳过） | `mage_ai/api/parsers/XxxParser.py` |
| `XxxMonitor` | ❌ | 错误包装与定制化错误呈现（默认使用 BaseMonitor） | `mage_ai/api/monitors/XxxMonitor.py` |

**新增资源零配置**：按命名约定创建上述文件，路由自动生效，无需修改任何路由表。

---

## 六、重要约束与设计要点

### 6.1 权限三重漏斗模型

Policy 授权按**粒度从粗到细**分为三层校验（注意：这是层级概念，不是严格的调用顺序）：

```
1. authorize_action() ── 漏斗 1：动作级（最粗）
   能否对该资源执行 CREATE/LIST/DETAIL/UPDATE/DELETE？
   规则来源: Policy.allow_actions()

   ├─ 2. authorize_query() ── 漏斗 2a：查询参数级
   │      能否按该条件过滤？(如 ?status=active&type=python)
   │      规则来源: Policy.allow_query()

   └─ 3. authorize_attributes() ── 漏斗 2b/3：字段级（最细）
          能否读/写该字段？(如读取 password_hash、修改 user.role)
          规则来源: Policy.allow_read() / allow_write()
```

**调用顺序与 Resource 加载的关系**（重要，不要与"粒度从粗到细"混淆）：

- **成员资源路径（DELETE/DETAIL/UPDATE）**：先 `process_member()` 加载 Resource → 再 `authorize_action()` 做第一层授权
- **集合创建路径（CREATE/LIST）**：先 `authorize_action()` 做第一层授权 → 再 `process_create/collection()` 调用 Resource

两条路径都经过相同的三层漏斗粒度，但 **Resource 调用与漏斗 1 的先后顺序相反**。详见 5.4 节的双路径生命周期图。

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
2. **类命名严格约定**：资源名复数 → 单数 → 驼峰，否则 `ModuleNotFoundError`（动态加载依赖命名约定）
3. **认证错误不抛异常**：OAuthMiddleware 仅设置 `request.error` 属性，不 raise，由 `execute_operation` 开头处读取渲染
4. **两条路径中 BEFORE Hooks 与 Resource 的顺序相反**：
   - DELETE/DETAIL/UPDATE：Resource.process_member() 加载 → BEFORE Hooks → authorize_action() 授权
   - CREATE/LIST：BEFORE Hooks → authorize_action() 授权 → Resource.process_create/collection()
   - 原因：DELETE/DETAIL/UPDATE 的授权条件和 Hooks 逻辑可能需要访问已加载对象的属性
5. **Policy/Parser 实例化参数差异**：路径 A 传入 `resource=已加载对象`，路径 B 传入 `resource=None`，写 Policy 时需注意是否依赖对象存在
6. **授权做两次**：
   - 业务操作前：WRITE 授权（CREATE/UPDATE）或 QUERY 授权（LIST/DETAIL/UPDATE）
   - 展示后：所有 action 统一做 `authorize_attributes(READ)` 字段级读过滤（Presenter 已转成 dict）
7. **DELETE 无 Parser**：DELETE 动作除了 authorize_action()，不调用任何 Parser/字段授权，需注意 Policy 的 allow_actions 规则是否足够
8. **异步同步混用**：`BaseOperation.execute()` 是 async，但 Resource 内部部分方法（文件操作、部分 DB 查询）仍用同步 I/O，注意长阻塞

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

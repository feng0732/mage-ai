# RBAC 权限流与多租户隔离深度分析

## 一、两种权限模式的核心差异

Mage AI 存在**两套独立的权限检查路径**，通过环境变量 `REQUIRE_USER_PERMISSIONS` 切换：

| 维度 | 旧模式（角色权限） | 新模式（细粒度权限） |
|------|-------------------|---------------------|
| 开关 | `REQUIRE_USER_PERMISSIONS=False` | `REQUIRE_USER_PERMISSIONS=True` |
| 核心入口 | `has_at_least_viewer_role()` 等 | `validate_condition_with_permissions()` |
| 实体隔离维度 | `Entity` 枚举（GLOBAL/PROJECT/PIPELINE） | `entity_name` 资源类型 + `entity_id` 资源实例 |
| 层级继承 | 有（PIPELINE → PROJECT → GLOBAL） | 无 Entity 层级，但有 entity_name 通配符 |
| 权限表达 | 位掩码 + 角色聚合 | 位掩码 + entity_name 匹配 + entity_id 精确匹配 |
| 核心文件 | `mage_ai/api/utils.py` | `mage_ai/api/policies/mixins/user_permissions.py` |

---

## 二、旧模式：角色权限路径（Entity 层级继承）

### 2.1 调用链

```
policy.has_at_least_viewer_role()
    ↓  [mage_ai/api/policies/BasePolicy.py L384-L396]
mage_ai/api/utils.py: has_at_least_viewer_role(user, entity, entity_id)
    ↓
user.get_access(entity, entity_id)
    ↓  [mage_ai/orchestration/db/models/oauth.py L98-L129]
    1. fetch_roles([user_id])          → 查询用户所有角色
    2. Role.fetch_permissions(role_ids) → 查询角色直接关联的权限
    3. Role.fetch_role_permissions(role_ids) → 查询角色通过 role_permission 关联的权限
    4. 遍历每个 role:
       role.get_access(entity, entity_id, permissions_for_role)
       ↓  [mage_ai/orchestration/db/models/oauth.py L376-L409]
       a. 过滤匹配 entity + entity_id 的权限
       b. 位或运算聚合所有匹配权限的 access 位
       c. 如果没找到匹配 → 调用 get_parent_access() 向上继承
```

### 2.2 Entity 层级与继承关系

**文件**: `mage_ai/orchestration/constants.py`

```python
class Entity(StrEnum):
    ANY = 'any'          # 仅用于权限评估，不存储到DB
    GLOBAL = 'global'    # 全局级别，跨所有项目
    PROJECT = 'project'  # 项目级别，主要租户边界
    PIPELINE = 'pipeline' # 流水线级别
```

**继承规则**（`Role.get_parent_access()` [oauth.py L411-L431]）:

```
PIPELINE 级权限缺失时
    → 向上继承 PROJECT 级权限（entity_id = get_project_uuid()）
    → 再缺失，向上继承 GLOBAL 级权限
    → 再缺失，返回 0（无权限）
```

**继承代码**:
```python
def get_parent_access(self, entity, permissions_for_role=None):
    if entity == Entity.PIPELINE:
        return self.get_access(
            Entity.PROJECT, get_project_uuid(),
            permissions_for_role=permissions_for_role,
        )
    elif entity == Entity.PROJECT:
        return self.get_access(
            Entity.GLOBAL,
            permissions_for_role=permissions_for_role,
        )
    else:
        return 0
```

### 2.3 实体匹配逻辑

**文件**: `mage_ai/orchestration/db/models/oauth.py` `Role.get_access()` [L390-L396]

```python
entity_permissions = list(filter(
    lambda perm: perm.entity == entity and           # entity 类型匹配
    (entity_id is None or perm.entity_id == entity_id),  # entity_id 精确匹配（或为空）
    arr,
))
```

匹配规则：
1. **类型匹配**：`perm.entity == entity`（必须同为 GLOBAL/PROJECT/PIPELINE）
2. **实例匹配**：
   - 如果 `entity_id is None` → 匹配所有同类型 entity 的权限（通配）
   - 否则 `perm.entity_id == entity_id` → 精确匹配具体实例

### 2.4 各 Policy 的 entity 定义

| Policy 类 | entity 逻辑 | 文件 |
|-----------|------------|------|
| BasePolicy | 默认返回 `(Entity.PROJECT, get_project_uuid())` | `mage_ai/api/policies/BasePolicy.py L68-L69` |
| PipelinePolicy | 有资源时返回 `(Entity.PIPELINE, pipeline.uuid)`，否则 `(Entity.PROJECT, project_uuid)` | `mage_ai/api/policies/PipelinePolicy.py L10-L15` |
| PipelineRunPolicy | 从 query 或资源中提取 pipeline_uuid，返回 `(Entity.PIPELINE, pipeline_uuid)` | `mage_ai/api/policies/PipelineRunPolicy.py L11-L21` |
| WidgetPolicy | 有 parent_model 时返回 `(Entity.PIPELINE, parent_model.uuid)` | `mage_ai/api/policies/WidgetPolicy.py L11-L16` |
| SessionPolicy | 返回 `(Entity.ANY, None)` | `mage_ai/api/policies/SessionPolicy.py L46-L47` |

### 2.5 失败分支（旧模式）

**文件**: `mage_ai/api/policies/BasePolicy.py`

失败发生在 `__validate_condition()` [L646-L681]，当所有 condition 函数都返回 False 时：

1. **无规则配置** (`authorize_action` L431-L435):
   ```python
   if not config:  # action_rule() 返回 None
       error = ApiError.UNAUTHORIZED_ACCESS
       error.message = 'The {action} action is disabled for {resource_name}.'
       raise ApiError(error)
   ```

2. **Scope 验证失败** (`__validate_scopes` L683-L707):
   - 未登录用户访问 `CLIENT_PRIVATE` scope 的资源 → 403
   - 已登录用户访问仅 `CLIENT_PUBLIC` scope 的资源 → 403

3. **条件验证全部失败** (`__validate_condition` L646-L681):
   - 遍历所有 condition 函数
   - 主条件全失败 → 尝试 `override_permission_conditions`
   - 全部失败 → 抛出 403，附带：
     - 指标埋点 (`api_error.unauthorized_access`)
     - DEBUG 模式下附加 debug 信息（action、attribute_operation、operation）

4. **特殊错误增强** (`BaseOperation.execute` L231-L258):
   - 403 错误且用户无任何项目权限 (`project_access == 0`)
   - 旧模式下修改错误消息为友好提示："请向管理员申请权限"

---

## 三、新模式：细粒度权限路径（entity_name 匹配）

### 3.1 调用链

```
policy.authorize_action(action)
    ↓  [BasePolicy.py L72-L78, REQUIRE_USER_PERMISSIONS=True]
action_rule_with_permissions(operation)
    ↓  [user_permissions.py L258-L265]
    返回 { OauthScope.CLIENT_PRIVATE: [ { condition: validate_condition_with_cache } ] }
    ↓
validate_condition_with_cache(policy, operation)
    ↓  [user_permissions.py L28-L57]
    1. load_cached_permission_authorization() → 检查缓存
    2. 缓存命中 → 直接返回
    3. 缓存未命中 → validate_condition_with_permissions()
    4. cache_permission_authorization() → 写入缓存
```

### 3.2 核心验证逻辑

**文件**: `mage_ai/api/policies/mixins/user_permissions.py` `validate_condition_with_permissions()` [L60-L253]

```
1. Owner 快速通道:
   任一 permission.access & PermissionAccess.OWNER → 直接返回 True

2. 并行验证所有权限 (asyncio.gather):
   对每个 permission 调用 __permission_grants_access()
   返回 (permission_granted, permission_disabled)

3. 结果聚合:
   authorized = 任一 permission_granted 为 True
   unauthorized = 任一 permission_disabled 为 True
   返回 authorized and not unauthorized （禁用优先）
```

### 3.3 __permission_grants_access 详细步骤

**文件**: `mage_ai/api/policies/mixins/user_permissions.py` [L87-L237]

```
步骤 1: access 空值检查
    permission.access is None → (False, False)

步骤 2: Owner 位检查
    permission.access & OWNER → (True, False)

步骤 3: entity_name 匹配检查 (correct_entity_name)
    三种匹配方式（OR 关系）：
    a. 精确匹配: permission.entity_name == 当前 entity_name
    b. 全部匹配: permission.entity_name == EntityName.ALL
    c. 排除保留实体: permission.entity_name == ALL_EXCEPT_RESERVED
       且 当前 entity_name 不在 RESERVED_ENTITY_NAMES 中
    
    不匹配 → (False, False)

步骤 4: entity_id 精确匹配
    如果 permission.entity_id is not None 且有 resource:
        从 ENTITY_NAME_ENTITY_ID_ATTRIBUTE_NAME_MAPPING 获取ID属性名
        （如 Pipeline 用 uuid，其他用 id）
        resource.id != permission.entity_id → (False, False)

步骤 5: 禁用位检查 (disable_access)
    permission.access & disable_access → (False, True) 标记禁用

步骤 6: 权限位扩展 (PERMISSION_ACCESS_WITH_MULTIPLE_ACCESS)
    例如 EDITOR 自动扩展为: EDITOR + VIEWER + LIST + DETAIL + READ + ...

步骤 7: ALL 权限检查
    permission_access & ALL → (True, False) 全部通过

步骤 8: DISABLE_OPERATION_ALL 检查
    permission_access & DISABLE_OPERATION_ALL → (False, True) 全部禁用

步骤 9: 操作级权限检查
    has_access_for_all_operations = access & OPERATION_ALL
    valid_for_operation = has_access_for_all_operations or (access & 具体操作位)

步骤 10: 属性级权限检查（如果是属性操作）
    a. 检查 QUERY_ALL / READ_ALL / WRITE_ALL
    b. 检查具体属性操作位（QUERY/READ/WRITE）
    c. 禁用检查: disable_access_for_attribute_operation + 具体属性在 disabled_attributes 中
    d. 白名单检查: 具体属性在 query_attributes/read_attributes/write_attributes 中

步骤 11: 特殊条件检查 (DISABLE_UNLESS_CONDITIONS)
    条件列表（AND 关系，全部满足才有效）：
    - HAS_NOTEBOOK_EDIT_ACCESS: DISABLE_NOTEBOOK_EDIT_ACCESS != 1
    - HAS_PIPELINE_EDIT_ACCESS: not is_disable_pipeline_edit_access()
    - USER_OWNS_ENTITY: 用户是资源的所有者

步骤 12: 最终判断
    valid_for_operation → (True, False)
    否则 → (False, False)
```

### 3.4 entity_name 与 entity 的关键区别

| 概念 | 类型 | 用途 | 层级继承 |
|------|------|------|---------|
| `Entity` (GLOBAL/PROJECT/PIPELINE) | 枚举 | 旧模式：权限作用的**层级范围** | 有，向上继承 |
| `entity_name` (Pipeline, Block, ...) | 枚举 | 新模式：权限作用的**资源类型** | 无层级，但有通配符 |

**重要结论**:
- 新模式**完全不使用** `Permission.entity` 字段（GLOBAL/PROJECT/PIPELINE 层级）
- 新模式只使用 `permission.entity_name`（资源类型）和 `permission.entity_id`（资源实例）
- 新模式下，所有 Permission 记录不论 entity 字段值为何，都会被加载和检查

### 3.5 资源类型匹配规则

**文件**: `mage_ai/authentication/permissions/constants.py`

```python
class EntityName(StrEnum):
    ALL = 'ALL'
    ALL_EXCEPT_RESERVED = 'ALL_EXCEPT_RESERVED'
    Pipeline = 'Pipeline'
    Block = 'Block'
    Workspace = 'Workspace'
    ...  # 共约 90+ 种资源类型

RESERVED_ENTITY_NAMES = [
    Oauth,
    OauthAccessToken,
    OauthApplication,
    Workspace,  # Workspace 是保留实体
]
```

匹配优先级（从高到低）：
1. 精确匹配当前资源类型 → 最具体
2. `ALL_EXCEPT_RESERVED` 通配符 → 排除保留实体
3. `ALL` 通配符 → 所有实体

### 3.6 失败分支（新模式）

新模式的失败发生在两个层面：

**层面 1: Policy 框架层（与旧模式共用）**
- 同旧模式的 1~3 点（无规则配置、Scope 失败、条件验证失败）
- 但条件函数是 `validate_condition_with_cache()` 而非 `has_at_least_viewer_role()`

**层面 2: 细粒度权限验证层**

| 失败场景 | 位置 | 返回值 |
|----------|------|--------|
| permission.access 为空 | `user_permissions.py L98-L99` | `(False, False)` |
| entity_name 不匹配 | `user_permissions.py L107-L115` | `(False, False)` |
| entity_id 不匹配 | `user_permissions.py L118-L127` | `(False, False)` |
| 操作权限被禁用（disable_access） | `user_permissions.py L130-L131` | `(False, True)` 禁用标记 |
| 属性权限被禁用（DISABLE_QUERY_ALL 等） | `user_permissions.py L172-L174` | `(False, True)` 禁用标记 |
| 具体属性在 disabled_attributes 中 | `user_permissions.py L185-L198` | `(False, True)` 禁用标记 |
| 操作位不匹配 | `user_permissions.py L154-L156` | 继续检查 |
| 属性位不匹配 | `user_permissions.py L158-L211` | 继续检查 |
| 特殊条件不满足 (DISABLE_UNLESS_CONDITIONS) | `user_permissions.py L213-L231` | valid_for_operation 被置 False |
| 所有权限都不通过且无禁用 | 聚合逻辑 | `authorized=False, unauthorized=False` → 最终 False |
| 任一权限标记禁用 | 聚合逻辑 | `unauthorized=True` → 最终 False（禁用优先） |

**禁用优先原则**: 只要有一个 permission 返回 `permission_disabled=True`，即使其他权限授予了访问，整体也会被拒绝。

---

## 四、权限继承关系对比详解

### 4.1 旧模式：纵向层级继承

```
用户请求: 访问 pipeline-abc (Entity.PIPELINE, entity_id="pipeline-abc")
                │
                ▼
    检查 PIPELINE 级权限（entity_id="pipeline-abc"）
        有匹配权限？→ 使用该权限的 access 位
                │ 无
                ▼
    向上继承：检查 PROJECT 级权限（entity_id=project_uuid）
        有匹配权限？→ 使用该权限的 access 位
                │ 无
                ▼
    向上继承：检查 GLOBAL 级权限（entity_id=None）
        有匹配权限？→ 使用该权限的 access 位
                │ 无
                ▼
         返回 0（无权限）
```

**继承特点**:
- 单向：从具体到抽象，自底向上
- 查找：找到即停止，取最具体层级的权限
- 聚合：每个层级内多个权限位通过位或运算合并

### 4.2 新模式：横向通配符匹配

```
用户请求: 访问 Pipeline "pipeline-abc" (entity_name="Pipeline", entity_id="pipeline-abc")
                │
                ▼
    ┌─────────────────────────────────────┐
    │ 并行检查用户的所有 Permission 记录    │
    └─────────────────────────────────────┘
                │
                ├── permission A: entity_name="Pipeline", entity_id="pipeline-abc"
                │     → 精确匹配 → 检查 access 位
                │
                ├── permission B: entity_name="Pipeline", entity_id=None
                │     → 类型匹配，ID 通配 → 检查 access 位
                │
                ├── permission C: entity_name="ALL_EXCEPT_RESERVED"
                │     → 通配匹配（Pipeline 不是保留实体）→ 检查 access 位
                │
                ├── permission D: entity_name="ALL"
                │     → 全匹配 → 检查 access 位
                │
                └── permission E: entity_name="Block"
                      → entity_name 不匹配 → 跳过
                │
                ▼
    结果聚合（禁用优先）:
        authorized = any(permission_granted)
        unauthorized = any(permission_disabled)
        return authorized and not unauthorized
```

**匹配特点**:
- 所有权限并行检查，没有优先级
- entity_name 有三级匹配：精确 → ALL_EXCEPT_RESERVED → ALL
- entity_id 有两种匹配：精确 → 空（通配）
- 最终用"或"逻辑聚合所有授权，用"或"逻辑聚合所有禁用
- 禁用优先级高于授权

---

## 五、多租户隔离机制对比

### 5.1 旧模式下的多租户

租户边界 = `Entity.PROJECT + project_uuid`

```
权限存储示例:

Permission 1:
  entity = Entity.PROJECT
  entity_id = "project-a-uuid"
  access = EDITOR
  → 用户在 project-a 是 Editor

Permission 2:
  entity = Entity.PROJECT
  entity_id = "project-b-uuid"
  access = VIEWER
  → 用户在 project-b 是 Viewer

Permission 3:
  entity = Entity.GLOBAL
  entity_id = None
  access = ADMIN
  → 用户在所有项目都是 Admin（全局权限）
```

**隔离效果**:
- 每个项目的权限独立
- 全局权限可以跨项目
- PIPELINE 级权限可以在项目内进一步细化

### 5.2 新模式下的多租户

新模式**没有显式的租户边界**（因为不使用 Entity.PROJECT 层级），但可以通过以下方式实现类似效果：

```
方式 1: 通过 entity_name + entity_id 控制具体资源
    Permission:
      entity_name = "Pipeline"
      entity_id = "pipeline-in-project-a"
      access = EDITOR
    → 只能操作项目A中的特定流水线

方式 2: 通过 entity_name 通配控制一类资源
    Permission:
      entity_name = "Pipeline"
      entity_id = None  # 空表示所有 Pipeline
      access = VIEWER
    → 可以查看所有 Pipeline（跨项目！）
```

**重要提醒**: 新模式下如果不做额外处理，权限默认是跨项目的。因为新模式不检查 `permission.entity` 字段，只检查 `entity_name` 和 `entity_id`。

---

## 六、失败分支详细对比

| 失败类型 | 旧模式 | 新模式 | 共同 |
|----------|--------|--------|------|
| 无 action 规则配置 | ✅ | ✅（自动生成规则，不会出现） | ✅ |
| Scope 验证失败 | ✅ | ✅ | ✅ |
| 条件函数返回 False | ✅ (has_at_least_... 等) | ✅ (validate_condition_...) | 机制相同，函数不同 |
| Entity 层级不匹配 | ✅ | ❌（不使用 Entity 层级） | - |
| entity_name 不匹配 | ❌ | ✅ | - |
| entity_id 不匹配 | ✅（但继承可绕过） | ✅（无继承，精确匹配） | 机制不同 |
| 禁用权限优先 | ❌（位或聚合，禁用位不会抵消授权位） | ✅（显式禁用优先逻辑） | - |
| 属性级禁用 | ❌（属性级只有 allow/deny） | ✅（disabled_attributes 白/黑名单） | - |
| DEBUG 调试信息 | ✅ | ✅ | ✅ |
| 指标埋点 | ✅ | ✅ | ✅ |
| 无项目权限友好提示 | ✅（旧模式特有） | ❌ | - |

### 6.1 旧模式失败流程图

```
policy.authorize_action(action)
    │
    ├─ is_owner()？→ 是 → ✅ 通过
    │
    ├─ action_rule 存在？→ 否 → ❌ 403 (action disabled)
    │
    ├─ scope 验证
    │   ├─ 未登录 & 非 public scope → ❌ 403 (invalid scope)
    │   └─ 已登录 & 非 private scope → ❌ 403 (invalid scope)
    │
    └─ 条件验证（遍历所有 condition）
        ├─ 任一条件返回 True → ✅ 通过
        └─ 全部返回 False → ❌ 403 (failed condition)
           ↓
           [BaseOperation]
           ├─ 403 + 无项目权限 → 修改错误消息
           ├─ 运行失败钩子
           └─ DEBUG 模式重新抛出
```

### 6.2 新模式失败流程图

```
validate_condition_with_permissions()
    │
    ├─ user 有 Owner 权限？→ 是 → ✅ 通过
    │
    └─ 并行检查所有 permission
        │
        ├─ permission.access 为空？→ 跳过
        ├─ entity_name 不匹配？→ 跳过
        ├─ entity_id 不匹配？→ 跳过
        │
        ├─ 操作被禁用 (disable_access)？→ 标记 unauthorized += 1
        ├─ 属性操作被禁用？→ 标记 unauthorized += 1
        │
        ├─ ALL 权限？→ 标记 authorized += 1
        ├─ OPERATION_ALL + 具体操作位？→ 标记 authorized += 1
        └─ 具体操作位匹配 + 属性位匹配？→ 标记 authorized += 1
    │
    └─ 结果聚合
        ├─ unauthorized > 0 → ❌ 失败（禁用优先）
        ├─ authorized > 0 → ✅ 通过
        └─ 都没有 → ❌ 失败（无匹配权限）
```

---

## 七、关键代码位置索引

### 7.1 旧模式（角色权限路径）

| 功能 | 文件相对路径 | 行号 |
|------|-------------|------|
| 角色检查入口 | `mage_ai/api/utils.py` | L33-L103 |
| is_owner | `mage_ai/api/utils.py` | L33-L40 |
| has_at_least_admin_role | `mage_ai/api/utils.py` | L43-L52 |
| has_at_least_editor_role | `mage_ai/api/utils.py` | L55-L65 |
| has_at_least_viewer_role | `mage_ai/api/utils.py` | L92-L103 |
| User.get_access | `mage_ai/orchestration/db/models/oauth.py` | L98-L129 |
| Role.get_access | `mage_ai/orchestration/db/models/oauth.py` | L376-L409 |
| Role.get_parent_access（继承逻辑） | `mage_ai/orchestration/db/models/oauth.py` | L411-L431 |
| Entity 枚举 | `mage_ai/orchestration/constants.py` | L21-L27 |

### 7.2 新模式（细粒度权限路径）

| 功能 | 文件相对路径 | 行号 |
|------|-------------|------|
| 核心验证函数 | `mage_ai/api/policies/mixins/user_permissions.py` | L60-L253 |
| 缓存包装函数 | `mage_ai/api/policies/mixins/user_permissions.py` | L28-L57 |
| __permission_grants_access | `mage_ai/api/policies/mixins/user_permissions.py` | L87-L237 |
| 权限加载与缓存 | `mage_ai/api/mixins/result_set.py` | L29-L79 |
| 授权结果缓存 | `mage_ai/api/mixins/result_set.py` | L81-L170 |
| entity_name_uuid | `mage_ai/api/mixins/result_set.py` | L22-L27 |
| EntityName 枚举 | `mage_ai/authentication/permissions/constants.py` | L6-L100 |
| PermissionAccess 位掩码 | `mage_ai/authentication/permissions/constants.py` | L139-L176 |
| 权限位扩展映射 | `mage_ai/authentication/permissions/constants.py` | L222-L227 |
| 保留实体列表 | `mage_ai/authentication/permissions/constants.py` | L103-L108 |

### 7.3 共用框架

| 功能 | 文件相对路径 | 行号 |
|------|-------------|------|
| BasePolicy 基类 | `mage_ai/api/policies/BasePolicy.py` | L38-L724 |
| authorize_action | `mage_ai/api/policies/BasePolicy.py` | L398-L435 |
| authorize_attribute | `mage_ai/api/policies/BasePolicy.py` | L437-L502 |
| authorize_query | `mage_ai/api/policies/BasePolicy.py` | L511-L581 |
| __validate_condition | `mage_ai/api/policies/BasePolicy.py` | L616-L681 |
| __validate_scopes | `mage_ai/api/policies/BasePolicy.py` | L683-L707 |
| BaseOperation.execute | `mage_ai/api/operations/base.py` | L77-L262 |
| 错误处理与钩子 | `mage_ai/api/operations/base.py` | L231-L258 |
| ApiError 定义 | `mage_ai/api/errors.py` | L7-L63 |

---

## 八、易混淆点总结

1. **Entity vs entity_name**:
   - `Entity` (GLOBAL/PROJECT/PIPELINE): 旧模式的层级范围概念，有纵向继承
   - `entity_name` (Pipeline/Block/...): 新模式的资源类型概念，有横向通配符

2. **entity_id 的两种含义**:
   - 旧模式：配合 `Entity` 枚举使用，表示实体范围的实例 ID（如 project uuid）
   - 新模式：配合 `entity_name` 使用，表示具体资源实例的 ID（如 pipeline uuid）

3. **两种模式下 "权限继承" 的区别**:
   - 旧模式：纵向层级继承（PIPELINE → PROJECT → GLOBAL）
   - 新模式：横向通配符扩展（具体类型 → ALL_EXCEPT_RESERVED → ALL）

4. **权限聚合逻辑**:
   - 旧模式：位或运算，权限只增不减（没有"禁用"概念）
   - 新模式：授权或 + 禁用或，禁用优先级高于授权

5. **多租户隔离**:
   - 旧模式：通过 `Entity.PROJECT + project_uuid` 天然隔离
   - 新模式：不使用 Entity 层级，需要通过 entity_id 或额外逻辑实现隔离

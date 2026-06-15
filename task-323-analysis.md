# 多租户与权限系统分析

## 一、系统架构总览

Mage AI 的权限系统采用**声明式策略模式 + 位掩码权限 + 基于实体(Entity)的多租户隔离**三层架构。系统没有显式的 `Tenant` 类，而是通过 `Entity` 枚举和 `entity_id` 字段实现租户级别的权限隔离。

### 核心模块依赖关系

```
API Request
    ↓
[BaseOperation.execute()]  ── 关键入口
    ↓
[Policy.authorize_action()]  ── 操作级权限
[Policy.authorize_query()]   ── 查询参数权限
[Policy.authorize_attribute()] ── 属性级权限
    ↓
[validate_condition_with_permissions()]  ── 核心编排
    ↓
[User.get_access()] / [Role.get_access()]  ── 权限计算
    ↓
DB: user_role → role_permission → permission
```

---

## 二、关键入口分析

### 2.1 API 操作主入口

**文件**: [base.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py)

所有 API 请求的权限检查起点是 `BaseOperation.execute()` [L77-L262](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py#L77-L262)。

#### 调用时序:

1. **创建/列表操作** (`__create_or_index()` [L485-L580](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py#L485-L580)):
   ```python
   policy = self.__policy_class()(None, self.user, **updated_options)
   await policy.authorize_action(self.action)        # L501
   await policy.authorize_query(self.query)          # L568
   await policy.authorize_attributes(WRITE, ...)     # L529 (CREATE only)
   ```

2. **删除/详情/更新操作** (`__delete_show_or_update()` [L582-L712](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py#L582-L712)):
   ```python
   policy = self.__policy_class()(res, self.user, **updated_options)
   await policy.authorize_action(self.action)        # L600
   await policy.authorize_query(self.query)          # L661
   await policy.authorize_attributes(WRITE, ...)     # L679 (UPDATE only)
   ```

3. **结果返回前验证** (`execute()` [L183-L188](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py#L183-L188)):
   ```python
   await policy.authorize_attributes(READ, resource_attributes, ...)
   ```

### 2.2 策略类动态加载

**文件**: [base.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py)

`__policy_class()` [L763-L769](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py#L763-L769) 动态加载对应资源的 Policy 类:
```python
def __policy_class(self):
    return getattr(
        importlib.import_module(
            'mage_ai.api.policies.{}Policy'.format(self.__classified_class())
        ),
        '{}Policy'.format(self.__classified_class()),
    )
```

例如: 访问 `/pipelines` → 加载 `PipelinePolicy` → 继承 `BasePolicy`。

---

## 三、内部编排逻辑

### 3.1 双模式权限系统

**文件**: [BasePolicy.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py)

系统支持两种权限检查模式，通过环境变量 `REQUIRE_USER_PERMISSIONS` 切换:

| 模式 | 触发条件 | 核心逻辑 |
|------|---------|---------|
| **旧模式** | `REQUIRE_USER_PERMISSIONS=False` | 基于 `action_rules` / `read_rules` / `write_rules` 配置 |
| **新模式** | `REQUIRE_USER_PERMISSIONS=True` | 基于数据库 Permission 表的细粒度权限控制 |

#### 模式切换点:
- `action_rule()` [L72-L78](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L72-L78): 调用 `action_rule_with_permissions()`
- `read_rule()` / `write_rule()` [L102-L123](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L102-L123): 调用 `attribute_rule_with_permissions()`

### 3.2 授权核心流程

**文件**: [BasePolicy.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py)

`authorize_action()` [L398-L435](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L398-L435) 执行流程:

```
1. 快速跳过检查:
   ├─ 未启用用户认证 + 非编辑禁用 + 是写操作 → 直接通过
   └─ 启用权限检查 + 是Owner → 直接通过

2. 获取规则配置: action_rule(action)
   └─ 无配置 → 抛出 UNAUTHORIZED_ACCESS

3. 验证 Scope: __validate_scopes()
   └─ 检查当前用户 scope 是否匹配规则配置

4. 验证条件: __validate_condition()
   ├─ 遍历条件列表，任一条件满足则通过
   ├─ 主条件不满足时尝试 override_permission_conditions
   └─ 全部失败 → 抛出 UNAUTHORIZED_ACCESS
```

### 3.3 细粒度权限验证（新模式）

**文件**: [user_permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/mixins/user_permissions.py)

`validate_condition_with_permissions()` [L60-L253](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/mixins/user_permissions.py#L60-L253) 是新模式的核心:

```
1. 加载用户权限: load_and_cache_user_permissions()

2. Owner 快速通道: 任一 permission 含 OWNER 位 → 返回 True

3. 并行验证所有权限 (asyncio.gather):
   对每个 permission 调用 __permission_grants_access():

   3a. 实体名称匹配检查 (correct_entity_name):
       - permission.entity_name == 当前实体名
       - 或 == EntityName.ALL
       - 或 == EntityName.ALL_EXCEPT_RESERVED 且非保留实体

   3b. 实体 ID 匹配检查:
       - 如果 permission.entity_id 有值，必须与资源 ID 匹配

   3c. 禁用检查 (disable_access):
       - 匹配到禁用权限 → 返回 (False, True) 标记禁用

   3d. 权限位扩展 (PERMISSION_ACCESS_WITH_MULTIPLE_ACCESS):
       - OWNER → 包含 ADMIN + EDITOR + VIEWER + ...
       - ADMIN → 包含 EDITOR + VIEWER + ...
       - EDITOR → 包含 VIEWER + ...
       - VIEWER → 包含 LIST + DETAIL + READ

   3e. 位掩码权限检查:
       - ALL → 全部通过
       - DISABLE_OPERATION_ALL → 全部禁用
       - 操作级权限: access & (OPERATION_ALL | 具体操作位)
       - 属性级权限: access & (QUERY_ALL/READ_ALL/WRITE_ALL | 具体属性位)

   3f. 特殊条件检查 (DISABLE_UNLESS_CONDITIONS):
       - HAS_NOTEBOOK_EDIT_ACCESS
       - HAS_PIPELINE_EDIT_ACCESS
       - USER_OWNS_ENTITY

4. 结果聚合:
   - 任一 permission_disabled → unauthorized=True
   - 任一 permission_granted → authorized=True
   - 最终返回: authorized and not unauthorized
```

### 3.4 权限缓存机制

**文件**: [result_set.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/mixins/result_set.py)

为避免重复查询数据库，系统实现了两层缓存:

1. **用户权限缓存** (`load_and_cache_user_permissions()` [L29-L79](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/mixins/result_set.py#L29-L79)):
   - Key: `__user_permissions`
   - 存储用户所有 Permission 记录列表

2. **授权结果缓存** (`cache_permission_authorization()` [L127-L170](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/mixins/result_set.py#L127-L170)):
   - Key 结构: `__operations` 或 `__attribute_operations` → entity_name → operation → [attribute_operation] → [resource_attribute]
   - 缓存布尔授权结果

### 3.5 角色与权限计算

**文件**: [oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/orchestration/db/models/oauth.py)

`User.get_access()` [L98-L129](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/orchestration/db/models/oauth.py#L98-L129):
```python
def get_access(self, entity, entity_id):
    roles = self.fetch_roles([self.id])                    # 1. 查询用户角色
    permissions = Role.fetch_permissions([r.id for r in roles])  # 2. 查询角色权限
    permissions_role = Role.fetch_role_permissions(...)    # 3. 查询角色权限关联
    for role in roles:
        access |= role.get_access(entity, entity_id, permissions_for_role)
    return access
```

**位掩码权限定义** ([constants.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/authentication/permissions/constants.py)):
```python
class PermissionAccess(IntEnum):
    OWNER = 1       # 2^0
    ADMIN = 2       # 2^1
    EDITOR = 4      # 2^2
    VIEWER = 8      # 2^3
    LIST = 16       # 2^4
    DETAIL = 32     # 2^5
    CREATE = 64     # 2^6
    UPDATE = 128    # 2^7
    DELETE = 512    # 2^9
    ...
```

---

## 四、多租户实现原理

### 4.1 Entity 作为租户边界

**文件**: [constants.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/orchestration/constants.py)

系统没有显式的 `Tenant` 类，而是通过 `Entity` 枚举实现多租户隔离:

```python
class Entity(StrEnum):
    ANY = 'any'          # 仅用于权限评估，不存储
    GLOBAL = 'global'    # 全局级别，跨所有项目
    PROJECT = 'project'  # 项目级别，主要租户边界
    PIPELINE = 'pipeline' # 流水线级别
```

### 4.2 项目 UUID 作为租户标识

**文件**: [repo_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/data_preparation/repo_manager.py)

`get_project_uuid()` [L389-L390](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/data_preparation/repo_manager.py#L389-L390) 返回当前项目的 UUID，作为 `Entity.PROJECT` 级别的 `entity_id`。

**BasePolicy 默认实体** ([BasePolicy.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L68-L69)):
```python
@property
def entity(self) -> Tuple[Union[Entity, None], Union[str, None]]:
    return Entity.PROJECT, get_project_uuid()
```

这意味着**默认所有权限检查都在当前项目(租户)范围内进行**。

### 4.3 权限的实体范围

每个 `Permission` 记录通过以下字段确定其作用域:

| 字段 | 说明 | 示例 |
|------|------|------|
| `entity` | 实体类型层级 | `Entity.PROJECT` |
| `entity_id` | 实体实例标识 | `project_uuid` 值 |
| `entity_name` | 资源类型名称 | `EntityName.Pipeline` |
| `entity_id` (Permission 表) | 具体资源 ID | 某个 pipeline 的 uuid |

### 4.4 实体名称匹配规则

**文件**: [user_permissions.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/mixins/user_permissions.py)

`__permission_grants_access()` [L107-L112](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/mixins/user_permissions.py#L107-L112):

```python
correct_entity_name = (
    permission.entity_name == entity_name or                    # 精确匹配
    permission.entity_name == EntityName.ALL or                  # 匹配所有
    (
        permission.entity_name == EntityName.ALL_EXCEPT_RESERVED and
        entity_name not in RESERVED_ENTITY_NAMES                 # 排除保留实体
    )
)
```

**保留实体** ([constants.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/authentication/permissions/constants.py#L103-L108)):
```python
RESERVED_ENTITY_NAMES = [
    EntityName.Oauth,
    EntityName.OauthAccessToken,
    EntityName.OauthApplication,
    EntityName.Workspace,  # Workspace 属于保留实体，不参与 ALL_EXCEPT_RESERVED
]
```

### 4.5 工作空间(Workspace)的角色

**文件**: [WorkspacePolicy.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/WorkspacePolicy.py)

Workspace 是一个特殊的保留实体:
- 仅 Owner 可创建/删除/更新
- Viewer 及以上可查看列表和详情
- 不作为主要的租户边界，主要用于管理计算资源配置

---

## 五、失败分支与错误处理

### 5.1 权限检查失败点汇总

| 失败场景 | 位置 | 错误类型 |
|----------|------|---------|
| 无 action 规则配置 | [BasePolicy.py L431-L435](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L431-L435) | `UNAUTHORIZED_ACCESS` 403 |
| 无 attribute 规则配置 | [BasePolicy.py L480-L490](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L480-L490) | `UNAUTHORIZED_ACCESS` 403 |
| 无 query 规则配置 | [BasePolicy.py L566-L571](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L566-L571) | `UNAUTHORIZED_ACCESS` 403 |
| Scope 验证失败（未登录访问私有） | [BasePolicy.py L689-L698](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L689-L698) | `UNAUTHORIZED_ACCESS` 403 |
| Scope 验证失败（已登录访问公有） | [BasePolicy.py L699-L707](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L699-L707) | `UNAUTHORIZED_ACCESS` 403 |
| 条件验证全部失败 | [BasePolicy.py L646-L681](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L646-L681) | `UNAUTHORIZED_ACCESS` 403 |
| 权限被禁用 (permission_disabled) | [user_permissions.py L243-L253](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/mixins/user_permissions.py#L243-L253) | 内部返回 False，触发上层 403 |

### 5.2 错误增强与埋点

**文件**: [BasePolicy.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py)

`__validate_condition()` 失败时的增强处理 [L646-L681](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/BasePolicy.py#L646-L681):

```python
if not validation:
    # 1. 构建错误消息
    message = f'Unauthorized {attribute_operation} access for {action} on {r_name}'
    
    # 2. 指标埋点
    increment(
        'api_error.unauthorized_access',
        tags={
            'action': action,
            'attribute_operation': attribute_operation,
            'policy_class_name': self.__class__.__name__,
            'operation': operation,
        }
    )
    
    # 3. DEBUG 模式附加调试信息
    if settings.DEBUG:
        error['debug'] = {
            'action': action,
            'attribute_operation': attribute_operation,
            'operation': operation,
        }
    
    raise ApiError(error)
```

### 5.3 特殊错误处理分支

**文件**: [base.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py)

`execute()` 的异常捕获 [L231-L258](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/operations/base.py#L231-L258):

```python
except ApiError as err:
    if (
        err.code == 403
        and self.user
        and self.user.project_access == 0  # 用户在当前项目无任何权限
        and not self.user.roles             # 旧系统也无角色
    ):
        if not REQUIRE_USER_PERMISSIONS:
            err.message = (
                'You don't have access to this project. '
                + 'Please ask an admin or owner for permissions.'
            )  # 友好提示用户申请权限
    
    # 运行失败钩子
    results_altered = self.__run_hooks_after(
        condition=HookCondition.FAILURE,
        error=self.__present_error(err),
        ...
    )
    
    if settings.DEBUG:
        raise err  # DEBUG 模式重新抛出，便于调试
```

### 5.4 错误定义

**文件**: [errors.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/errors.py)

```python
class ApiError(Exception):
    UNAUTHORIZED_ACCESS = dict(
        code=403,
        message='Unauthorized access.',
        type='unauthorized_access',
    )
    INVALID_API_KEY = dict(code=401, ...)
    EXPIRED_OAUTH_TOKEN = dict(code=401, ...)
    RESOURCE_NOT_FOUND = dict(code=404, ...)
```

---

## 六、数据模型与关联

### 6.1 核心表结构

**文件**: [oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/orchestration/db/models/oauth.py)

```
user (用户表)
├── id (PK)
├── _owner (Boolean, 全局所有者标记)
├── roles (Integer, 旧系统位掩码角色)
└── roles_new (M:N → user_role → role)

role (角色表)
├── id (PK)
├── name (唯一，如 "Owner", "Admin")
├── user_id (创建者)
├── permissions (1:N → permission)
├── role_permissions (M:N → role_permission → permission)
└── users (M:N → user_role → user)

permission (权限表)
├── id (PK)
├── entity (Entity 枚举: GLOBAL/PROJECT/PIPELINE)
├── entity_id (实体实例ID)
├── entity_name (EntityName 枚举: Pipeline, Block 等)
├── entity_type (子类型: BlockType/PipelineType)
├── access (Integer 位掩码)
├── role_id (FK → role)
├── user_id (创建者)
├── options (JSON: conditions, query_attributes 等)
├── role (N:1 → role)
└── roles (M:N → role_permission → role)

user_role (用户-角色关联表)
├── user_id (FK → user)
└── role_id (FK → role)

role_permission (角色-权限关联表)
├── role_id (FK → role)
├── permission_id (FK → permission)
└── user_id (创建者)
```

### 6.2 默认角色与权限初始化

**文件**: [oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/orchestration/db/models/oauth.py)

`Role.create_default_roles()` [L318-L369](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/orchestration/db/models/oauth.py#L318-L369):
1. 调用 `Permission.create_default_permissions()` 创建 OWNER/ADMIN/EDITOR/VIEWER 四个基础权限
2. 创建对应角色并关联权限

默认角色权限映射:
| 角色 | access 位 | 包含权限 |
|------|-----------|---------|
| Owner | 1 | OWNER → ADMIN + EDITOR + VIEWER + 所有操作 |
| Admin | 2 | ADMIN → EDITOR + VIEWER + 所有操作 |
| Editor | 4 | EDITOR → VIEWER + LIST/DETAIL/CREATE/UPDATE/DELETE |
| Viewer | 8 | VIEWER → LIST + DETAIL + READ |

---

## 七、策略定义示例

**文件**: [WorkspacePolicy.py](file:///d:/fz/0601/solo-dogfeeding/code/323-mage-ai/mage_ai/api/policies/WorkspacePolicy.py)

```python
class WorkspacePolicy(BasePolicy):
    pass

# 1. 操作权限
WorkspacePolicy.allow_actions(
    [constants.DETAIL, constants.LIST],
    scopes=[OauthScope.CLIENT_PRIVATE],
    condition=lambda policy: policy.has_at_least_viewer_role()
)
WorkspacePolicy.allow_actions(
    [constants.CREATE, constants.DELETE, constants.UPDATE],
    scopes=[OauthScope.CLIENT_PRIVATE],
    condition=lambda policy: policy.is_owner()
)

# 2. 读属性权限
WorkspacePolicy.allow_read(
    WorkspacePresenter.default_attributes,
    scopes=[OauthScope.CLIENT_PRIVATE],
    on_action=[constants.DETAIL, constants.LIST],
    condition=lambda policy: policy.has_at_least_viewer_role()
)

# 3. 写属性权限
WorkspacePolicy.allow_write(
    ['name', 'cluster_type', ...],
    scopes=[OauthScope.CLIENT_PRIVATE],
    on_action=[constants.CREATE, constants.UPDATE],
    condition=lambda policy: policy.is_owner()
)

# 4. 查询参数权限
WorkspacePolicy.allow_query(
    ['namespace[]', 'cluster_type', 'user_id'],
    scopes=[OauthScope.CLIENT_PRIVATE],
    on_action=[constants.LIST],
    condition=lambda policy: policy.has_at_least_viewer_role()
)
```

---

## 八、核心配置开关

| 环境变量 | 作用 |
|----------|------|
| `REQUIRE_USER_AUTHENTICATION` | 是否启用用户认证 |
| `REQUIRE_USER_PERMISSIONS` | 是否启用细粒度权限模式（新模式） |
| `DISABLE_NOTEBOOK_EDIT_ACCESS` | 是否禁用 Notebook 编辑权限 |

这些开关在 `BasePolicy` 的快速跳过检查中被频繁使用，例如:
```python
if not REQUIRE_USER_AUTHENTICATION and not DISABLE_NOTEBOOK_EDIT_ACCESS and action in [CREATE, DELETE, UPDATE]:
    return True  # 直接跳过权限检查
```

---

## 九、关键调用链总结

### 9.1 创建 Pipeline 的权限检查流程

```
POST /pipelines
    ↓
BaseOperation.execute()
    ├─ __create_or_index()
    │   ├─ policy = PipelinePolicy(None, current_user)
    │   ├─ await policy.authorize_action(CREATE)
    │   │   ├─ is_owner() → 否
    │   │   ├─ action_rule(CREATE) → 获取规则
    │   │   ├─ __validate_scopes() → scope 验证
    │   │   └─ __validate_condition()
    │   │       └─ validate_condition_with_cache()
    │   │           └─ validate_condition_with_permissions()
    │   │               ├─ load_and_cache_user_permissions()
    │   │               ├─ 检查 Owner 权限 → 否
    │   │               ├─ 遍历用户权限 → 检查 CREATE 位
    │   │               └─ 返回 True/False
    │   ├─ await policy.authorize_attributes(WRITE, payload.keys())
    │   └─ PipelineResource.process_create()
    └─ 结果读取验证: authorize_attributes(READ, ...)
```

### 9.2 权限位判断的数学原理

使用位运算进行高效权限判断:
```python
# 检查是否有 EDITOR 权限 (位 2，值为 4)
if user.get_access(...) & Permission.Access.EDITOR != 0:
    # 有权限

# 组合多个权限
access = Permission.Access.LIST | Permission.Access.DETAIL  # 16 | 32 = 48

# 扩展权限位（EDITOR 自动包含 VIEWER 的权限）
access = Permission.add_accesses([LIST, DETAIL, CREATE, UPDATE, DELETE, EDITOR])
```

---

## 十、设计特点与注意事项

1. **双重模式兼容**: 旧模式（基于规则配置）和新模式（基于数据库权限）并存，通过 `REQUIRE_USER_PERMISSIONS` 切换

2. **三层权限检查**:
   - 操作级 (authorize_action): 能否执行 CREATE/UPDATE 等
   - 查询级 (authorize_query): 能否使用特定查询参数
   - 属性级 (authorize_attribute): 能否读/写特定字段

3. **缓存优化**: 用户权限和授权结果两级缓存，避免重复数据库查询

4. **多租户隔离**: 通过 `Entity.PROJECT + project_uuid` 实现项目级隔离，权限默认限制在当前项目内

5. **禁用优先原则**: 权限验证时 `permission_disabled` 优先级高于 `permission_granted`，只要有一个权限标记禁用，整体就失败

6. **位掩码性能**: 使用整数位掩码存储权限，计算高效，但可读性较差

7. **保留实体**: Workspace、OAuth 相关实体属于保留实体，不参与 `ALL_EXCEPT_RESERVED` 通配

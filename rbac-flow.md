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

### 3.4 资源对象为空时的 entity_id 匹配（LIST/CREATE 场景）

#### 3.4.1 核心判断逻辑

**文件**: `mage_ai/api/policies/mixins/user_permissions.py` [L117-L127]

```python
# If the permission has an entity_id, check to see if it matches.
if permission.entity_id is not None and resource:
    id_attribute_name = 'id'
    if entity_name in ENTITY_NAME_ENTITY_ID_ATTRIBUTE_NAME_MAPPING:
        id_attribute_name = ENTITY_NAME_ENTITY_ID_ATTRIBUTE_NAME_MAPPING[entity_name]

    if not hasattr(resource, id_attribute_name) or \
            str(permission.entity_id) != str(getattr(resource, id_attribute_name)):
        return (False, False)
```

**关键条件**: `permission.entity_id is not None and resource`

这是一个 `AND` 逻辑，**两个条件同时满足才会进行 entity_id 检查**。这意味着：

| 场景 | permission.entity_id | resource | entity_id 检查 | 权限是否参与判断 |
|------|---------------------|----------|---------------|-----------------|
| 详情/更新/删除 | 有值 | 存在 | ✅ 执行精确匹配 | 匹配则参与，不匹配则跳过 |
| 详情/更新/删除 | None | 存在 | ❌ 跳过 | ✅ 参与（通配） |
| **列表/创建** | 有值 | **None** | **❌ 跳过** | **✅ 参与（即使 entity_id 不匹配）** |
| 列表/创建 | None | None | ❌ 跳过 | ✅ 参与 |

#### 3.4.2 LIST 操作的两阶段权限检查

**文件**: `mage_ai/api/operations/base.py`

LIST 操作有两个独立的权限检查阶段：

**阶段 1：操作级检查（获取列表前）** [L500-L501]
```python
policy = self.__policy_class()(None, self.user, **updated_options)  # resource=None
await policy.authorize_action(self.action)  # LIST 操作级检查
```
- resource = None
- entity_id 检查被跳过
- 带具体 entity_id 的权限也会参与授权判断
- 结果：决定"能不能调用列表接口"

**阶段 2：属性级检查（返回结果前，逐个资源）** [L145-L188]
```python
for idx, res in enumerate(results):
    policy = self.__policy_class()(res, self.user, **updated_options)  # resource=具体资源
    await policy.authorize_attributes(READ, resource_attributes, ...)
```
- resource = 具体资源对象
- entity_id 检查生效
- 每个资源独立判断
- 结果：决定"能不能读取这个资源的属性"
- ⚠️ 任何一个资源检查失败 → 整个请求抛出 403

#### 3.4.3 CREATE 操作的权限检查

**阶段 1：操作级检查（创建前）** [L500-L501]
- resource = None
- entity_id 检查被跳过

**阶段 2：写属性检查（创建前）** [L515-L529]
- resource = None（因为还没创建出来）
- entity_id 检查被跳过
- 检查 payload 中的字段是否可写

#### 3.4.4 对跨项目隔离的影响

由于新模式**不使用** `Permission.entity` 字段（GLOBAL/PROJECT/PIPELINE 层级），且 resource=None 时 entity_id 检查被跳过，导致：

1. **LIST 操作级检查无法实现项目隔离**：
   - 用户在项目 A 中有一个特定 pipeline 的权限（entity_id="pipeline-in-proj-A"）
   - 用户切换到项目 B，调用 LIST /pipelines
   - 阶段 1 操作级检查：resource=None → entity_id 检查跳过 → 权限有效 → ✅ 通过
   - 阶段 2 属性级检查：如果项目 B 中没有 entity_id 匹配的 pipeline → 所有 pipeline 都不匹配 → ❌ 403

2. **CREATE 操作完全无法通过 entity_id 控制**：
   - CREATE 时 resource 始终为 None
   - entity_id 检查永远被跳过
   - 只要有 entity_name 级别的 CREATE 权限就可以创建

3. **实际隔离效果**：
   - 操作级（能不能访问接口）：❌ 无隔离
   - 属性级（能不能读具体资源的字段）：✅ 有隔离，但失败时整体 403，而非过滤掉无权限资源

---

### 3.5 授权缓存机制与跨资源污染

#### 3.5.1 缓存结构

**文件**: `mage_ai/api/mixins/result_set.py` [L81-L170]

缓存存储在 `ResultSet.context.data` 中，key 结构如下：

```
操作级缓存：
  operations → entity_name → operation → authorized(bool)

属性级缓存：
  attribute_operations → entity_name → operation → attribute_operation_type → resource_attribute → authorized(bool)
```

**关键观察**：缓存 key 中**不包含 entity_id**！

#### 3.5.2 缓存读写流程

**文件**: `mage_ai/api/policies/mixins/user_permissions.py` [L28-L57]

```python
async def validate_condition_with_cache(policy, operation, ...):
    # 1. 尝试读取缓存
    result = await (policy.resource or policy).load_cached_permission_authorization(...)
    if result is not None:
        return result  # 缓存命中，直接返回，不做任何检查

    # 2. 缓存未命中，执行完整检查（包括 entity_id 匹配）
    authorized = await validate_condition_with_permissions(policy, ...)

    # 3. 写入缓存
    await (policy.resource or policy).cache_permission_authorization(authorized, ...)

    return authorized
```

#### 3.5.3 跨资源污染问题

由于同一个 ResultSet 中的所有资源共享同一个缓存（`BasePolicy.result_set()` [BasePolicy.py L610-L614] 返回资源所属的 result_set），且缓存 key 不含 entity_id，会出现以下问题：

**场景示例**：用户只有 pipeline-a 的权限，调用 LIST /pipelines 返回 [pipeline-a, pipeline-b, pipeline-c]

```
遍历第 1 个资源 pipeline-a:
  ├─ 查缓存：operations → Pipeline → LIST → 无
  ├─ 执行完整检查:
  │   └─ entity_id 匹配（pipeline-a == pipeline-a）→ ✅ 授权
  └─ 写入缓存：operations → Pipeline → LIST → True

遍历第 2 个资源 pipeline-b:
  ├─ 查缓存：operations → Pipeline → LIST → True（命中！）
  └─ 直接返回 True ❌（跳过了 entity_id 检查）

遍历第 3 个资源 pipeline-c:
  ├─ 查缓存：operations → Pipeline → LIST → True（命中！）
  └─ 直接返回 True ❌（跳过了 entity_id 检查）
```

**结果**：原本只有 pipeline-a 权限的用户，会被认为对所有 pipeline 都有权限。

#### 3.5.4 对权限粒度控制的影响（重要纠正）

缓存污染会削弱细粒度权限控制，但**不会导致跨项目数据泄漏**：

1. **放大效应（同项目内）**：只要有一个资源匹配了权限，同类型的所有资源都会被认为有权限
2. **作用域限制**：
   - ✅ 仅在同一请求内生效
   - ✅ 仅作用于当前项目的数据（数据层已过滤）
   - ❌ 不会跨请求（每个请求有独立的 ResultSet）
   - ❌ 不会跨项目（数据查询层 repo_path 过滤是独立防线）
3. **属性级同样受影响**：属性级缓存的 key 也不含 entity_id，同样存在污染问题
4. **关键澄清**："项目间泄漏"只有在数据查询层也出现缺陷时才会发生，默认配置下不会泄漏数据

---

### 3.6 禁用优先原则与跨项目影响

#### 3.6.1 禁用优先的聚合逻辑

**文件**: `mage_ai/api/policies/mixins/user_permissions.py` [L243-L253]

```python
authorized = False
unauthorized = False
for permission_granted, permission_disabled in permission_access_arr:
    if permission_granted:
        authorized = True
    if permission_disabled:
        unauthorized = True

return authorized and not unauthorized  # 禁用优先
```

**禁用优先原则**：只要有任何一个权限标记为禁用（`permission_disabled=True`），即使其他权限授予了访问，整体也会被拒绝。

#### 3.6.2 禁用权限的三种来源

| 禁用类型 | 触发条件 | 代码位置 |
|----------|---------|---------|
| 操作级禁用 | `permission.access & disable_access` | `user_permissions.py L130-L131` |
| 属性操作全禁用 | `permission.access & DISABLE_QUERY_ALL` 等 | `user_permissions.py L172-L174` |
| 具体属性禁用 | `attribute in permission.access_options['disabled_attributes']` | `user_permissions.py L185-L198` |

#### 3.6.3 禁用权限的影响范围（重要纠正）

由于新模式不区分 Entity 层级，且 entity_id 在 resource=None 时不检查，禁用权限可能产生**跨资源类型**的影响，但**不会跨项目泄漏数据**：

**场景示例**：
- 项目 A 中，用户有一个全局 Viewer 权限（entity_name=ALL, access=VIEWER）
- 管理员设置了一个禁用权限（entity_name=Pipeline, entity_id=None, access=DISABLE_OPERATION_ALL）
- 用户在项目 A 调用 LIST /pipelines

```
阶段 1 操作级检查（resource=None）:
  ├─ Permission 1: entity_name=ALL, access=VIEWER
  │   └─ entity_name 匹配 → LIST 位扩展 → ✅ 授权
  └─ Permission 2: entity_name=Pipeline, access=DISABLE_OPERATION_ALL
      └─ resource=None → entity_id 检查跳过 → ❌ 禁用
结果: authorized=True, unauthorized=True → ❌ 被禁用
```

**影响分析**：
- ✅ 不会导致其他项目的数据被返回（数据层 repo_path 过滤）
- ⚠️ 可能错误地拒绝当前项目的合法访问（禁用权限影响范围过大）
- ⚠️ 禁用权限不区分项目，可能影响所有项目中的 Pipeline 访问
- ❌ 不会"泄漏"数据，只会"误伤"合法访问

**关键澄清**：这里的"跨项目影响"是指禁用权限可能在多个项目中生效（因为权限加载不按项目过滤），但不会导致项目 A 的数据返回到项目 B 的请求中。

#### 3.6.4 禁用权限与缓存的交互

禁用权限同样会被缓存，且由于缓存 key 不含 entity_id：

1. 如果第一个检查的资源触发了禁用 → 缓存写入 `authorized=False`
2. 后续所有同类型资源都会读取到 `False` → 全部被拒绝
3. 反之，如果第一个资源授权通过 → 缓存 `True` → 后续资源都通过（即使有禁用权限）

**结果**：禁用优先的效果取决于遍历顺序，具有不确定性。

---

### 3.7 LIST 操作完整权限时序

以用户只有 pipeline-a 的权限（entity_id 精确匹配）为例，完整时序如下：

```
用户请求: GET /pipelines （LIST 操作）
│
├─ 阶段 1: 操作级检查 [base.py L500-L501]
│   ├─ policy = PipelinePolicy(None, user)  # resource=None
│   └─ authorize_action(LIST)
│       └─ validate_condition_with_cache()
│           ├─ 查缓存 → 无
│           ├─ validate_condition_with_permissions()
│           │   ├─ 遍历用户所有权限
│           │   ├─ permission(entity_id="pipeline-a"):
│           │   │   ├─ entity_name 匹配 Pipeline → ✅
│           │   │   ├─ resource=None → entity_id 检查跳过 → ✅
│           │   │   └─ LIST 位匹配 → (True, False)
│           │   └─ 聚合: authorized=True, unauthorized=False → True
│           └─ 缓存写入: operations → Pipeline → LIST → True
│   结果: ✅ 通过操作级检查
│
├─ 阶段 2: 查询级检查 [base.py L568]
│   └─ authorize_query(query_params)
│       └─ （类似操作级，resource=None，entity_id 检查跳过）
│
├─ 阶段 3: 获取数据
│   └─ process_collection() → 返回 [pipeline-a, pipeline-b, pipeline-c]
│
└─ 阶段 4: 属性级检查（逐个资源）[base.py L145-L188]
    │
    ├─ 资源 1: pipeline-a
    │   ├─ policy = PipelinePolicy(pipeline-a, user)  # resource=pipeline-a
    │   └─ authorize_attributes(READ, attributes)
    │       └─ 对每个属性调用 authorize_attribute()
    │           └─ validate_condition_with_cache()
    │               ├─ 查缓存 → 无（属性级缓存是空的）
    │               ├─ validate_condition_with_permissions()
    │               │   ├─ entity_name 匹配 → ✅
    │               │   ├─ entity_id 匹配（pipeline-a）→ ✅
    │               │   └─ READ 位匹配 → ✅ 通过
    │               └─ 缓存写入: attribute_operations → Pipeline → LIST → READ → attr → True
    │   结果: ✅ 通过
    │
    ├─ 资源 2: pipeline-b
    │   ├─ policy = PipelinePolicy(pipeline-b, user)  # resource=pipeline-b
    │   └─ authorize_attributes(READ, attributes)
    │       └─ 对每个属性调用 authorize_attribute()
    │           └─ validate_condition_with_cache()
    │               ├─ 查缓存: attribute_operations → Pipeline → LIST → READ → attr → True
    │               └─ 直接返回 True ⚠️（缓存命中，跳过 entity_id 检查）
    │   结果: ❌ 本应失败，但因缓存污染而通过
    │
    └─ 资源 3: pipeline-c
        └─ 同样因缓存污染而通过 ⚠️
```

**最终结论（重要校准）**：在新模式下，由于 entity_id 检查在 resource=None 时被跳过，且授权缓存不含 entity_id 维度，细粒度的 entity_id 权限在 LIST 操作中基本起不到隔离作用。但需要明确：
- ✅ **不会跨项目泄漏数据**（数据层 repo_path 过滤是独立可靠的防线）
- ⚠️ **同项目内越权访问**（用户可以看到当前项目中原本无权限的资源）
- ❌ **LIST 操作级过度放行**（本应拒绝的用户可以调用接口）

---

### 3.8 entity_name 与 entity 的关键区别

| 概念 | 类型 | 用途 | 层级继承 | 缓存是否区分 |
|------|------|------|---------|-------------|
| `Entity` (GLOBAL/PROJECT/PIPELINE) | 枚举 | 旧模式：权限作用的**层级范围** | 有，向上继承 | 不适用 |
| `entity_name` (Pipeline, Block, ...) | 枚举 | 新模式：权限作用的**资源类型** | 无层级，但有通配符 | ✅ 区分 |
| `entity_id` (资源实例 ID) | 字符串 | 新模式：权限作用的**具体资源** | 无，但 resource=None 时跳过 | ❌ 不区分 |

**重要结论（校准版）**:
- 新模式**完全不使用** `Permission.entity` 字段（GLOBAL/PROJECT/PIPELINE 层级）进行权限验证
- 新模式只使用 `permission.entity_name`（资源类型）和 `permission.entity_id`（资源实例）进行匹配
- 新模式下，所有 Permission 记录不论 entity 字段值为何，都会被加载和检查
- entity_id 精确匹配在操作级和列表场景下基本失效，仅在单资源操作（DETAIL/UPDATE/DELETE）中有效
- 缓存进一步削弱了 entity_id 的隔离作用
- ✅ **关键澄清**：虽然权限层不感知项目，但**数据层的 repo_path 过滤**提供了可靠的项目隔离，默认配置下不会发生跨项目数据泄漏

### 3.9 资源类型匹配规则

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

### 3.10 失败分支（新模式）

新模式的失败发生在三个层面：

**层面 1: Policy 框架层（与旧模式共用）**
- 同旧模式的 1~3 点（无规则配置、Scope 失败、条件验证失败）
- 但条件函数是 `validate_condition_with_cache()` 而非 `has_at_least_viewer_role()`

**层面 2: 细粒度权限验证层**

| 失败场景 | 位置 | 返回值 |
|----------|------|--------|
| permission.access 为空 | `user_permissions.py L98-L99` | `(False, False)` |
| entity_name 不匹配 | `user_permissions.py L107-L115` | `(False, False)` |
| entity_id 不匹配（resource 存在时） | `user_permissions.py L118-L127` | `(False, False)` |
| 操作权限被禁用（disable_access） | `user_permissions.py L130-L131` | `(False, True)` 禁用标记 |
| 属性权限被禁用（DISABLE_QUERY_ALL 等） | `user_permissions.py L172-L174` | `(False, True)` 禁用标记 |
| 具体属性在 disabled_attributes 中 | `user_permissions.py L185-L198` | `(False, True)` 禁用标记 |
| 操作位不匹配 | `user_permissions.py L154-L156` | 继续检查 |
| 属性位不匹配 | `user_permissions.py L158-L211` | 继续检查 |
| 特殊条件不满足 (DISABLE_UNLESS_CONDITIONS) | `user_permissions.py L213-L231` | valid_for_operation 被置 False |
| 所有权限都不通过且无禁用 | 聚合逻辑 | `authorized=False, unauthorized=False` → 最终 False |
| 任一权限标记禁用 | 聚合逻辑 | `unauthorized=True` → 最终 False（禁用优先） |

**禁用优先原则**: 只要有一个 permission 返回 `permission_disabled=True`，即使其他权限授予了访问，整体也会被拒绝。

**层面 3: 缓存层（隐性失败/绕过）**

缓存可能导致权限检查被绕过，产生非预期的失败或通过：

| 缓存相关场景 | 结果 | 原因 |
|-------------|------|------|
| 第一个资源授权通过 → 后续资源缓存命中 | 全部通过 | 缓存 key 不含 entity_id |
| 第一个资源被禁用 → 后续资源缓存命中 | 全部失败 | 禁用结果也会被缓存 |
| resource=None 时缓存的结果应用到有 resource 的场景 | 结果不确定 | 两种场景 entity_id 检查行为不同 |
| ~~不同项目共享 ResultSet~~ | ~~跨项目权限泄漏~~ | ✅ **修正**：ResultSet 是请求级的，不会跨项目共享；数据层 repo_path 过滤防止跨项目数据泄漏 |

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

## 五、资源类型与数据层隔离分类

数据层的项目隔离并非对所有资源一视同仁。不同资源类型的数据存储方式、查询路径、隔离机制差异很大。以下按六大类别分别说明。

### 5.1 第一类：文件系统存储 + repo_path 过滤

**代表资源**：Pipeline, Block, File, Widget, Variable, Schedule（YAML文件）, CustomTemplate 等

**数据存储**：文件系统，数据物理分布在各项目目录下

**隔离机制**：通过 `get_repo_path()` 确定当前项目根路径，所有文件读写都限制在该目录下

**代码位置**：
- `mage_ai/api/resources/PipelineResource.py` [L204]: `Pipeline.get_all_pipelines(repo_path=repo_path)`
- `mage_ai/settings/repo.py` [L34-L64]: `get_repo_path()` 确定当前项目路径
- `mage_ai/server/server.py` [L784]: 服务器启动时 `set_repo_path(project)`

**特点**：
- ✅ **物理隔离最可靠**：数据文件本身就在不同目录下
- ✅ 权限系统失效也不会泄漏其他项目数据
- ⚠️ 依赖 repo_path 的正确设置，如果路径被篡改可能越界
- ⚠️ `repo_path is None` 的 OR 条件可能泄漏无主记录

**典型查询**：
```python
# PipelineResource.collection 中
repo_path = get_repo_path()
pipeline_uuids = Pipeline.get_all_pipelines(repo_path=repo_path)
```

**策略层 entity 定义**：通常定义了 `entity` 属性，返回 `Entity.PIPELINE, pipeline_uuid` 或 `Entity.PROJECT, project_uuid`

---

### 5.2 第二类：数据库存储 + repo_path 字段过滤（repo_query）

**代表资源**：PipelineSchedule, Secret, Backfill 等

**数据存储**：数据库表，每条记录有 `repo_path` / `repo_name` 字段

**隔离机制**：通过 `repo_query` 类属性自动过滤，SQL 条件为 `repo_path == current_repo_path OR repo_path IS NULL`

**代码位置**：
- `mage_ai/orchestration/db/models/schedules.py` [L112-L121]: `PipelineSchedule.repo_query`
- `mage_ai/orchestration/db/models/secrets.py` [L19-L26]: `Secret.repo_query`
- `mage_ai/api/resources/PipelineScheduleResource.py` [L80-L83]: 非项目平台模式下使用 `repo_query`

**特点**：
- ✅ 数据库级过滤，相对可靠
- ⚠️ `repo_path IS NULL` 的 OR 条件会包含"全局"记录
- ⚠️ 如果忘记使用 `repo_query` 而直接用 `.query`，会返回全部数据

**典型查询**：
```python
@classproperty
def repo_query(cls):
    return cls.query.filter(
        or_(
            PipelineSchedule.repo_path == get_repo_path(),
            PipelineSchedule.repo_path.is_(None),
        )
    )
```

**策略层 entity 定义**：部分资源（如 PipelineRun）定义了 `entity` 属性，部分没有。

---

### 5.3 第三类：数据库存储 + 关联表间接隔离

**代表资源**：PipelineRun, BlockRun, Output 等运行时数据

**数据存储**：数据库表，通过 `pipeline_schedule_id` 或 `pipeline_uuid` 关联到项目

**隔离机制**：不直接按 repo_path 过滤，而是先获取当前项目的 schedule/pipeline ID 列表，再用 `IN` 子查询过滤

**代码位置**：
- `mage_ai/api/resources/PipelineRunResource.py` [L99, L135-L136]:
  ```python
  repo_pipeline_schedule_ids = [s.id for s in PipelineSchedule.repo_query]
  query = query.filter(PipelineRun.pipeline_schedule_id.in_(repo_pipeline_schedule_ids))
  ```

**特点**：
- ✅ 间接隔离，依赖上游的 repo_query 正确性
- ⚠️ 多一层关联，性能较差
- ⚠️ 存在 `include_all_pipeline_schedules` 参数可以绕过过滤
- ⚠️ 如果上游 schedule 数据被污染，下游 run 数据也会泄漏

**典型绕过风险**：
```python
include_all_pipeline_schedules = query_arg.get('include_all_pipeline_schedules', [None])
if not include_all_pipeline_schedules:
    query = query.filter(PipelineRun.pipeline_schedule_id.in_(repo_pipeline_schedule_ids))
# 如果传了 include_all_pipeline_schedules 参数，就不过滤了 ⚠️
```

---

### 5.4 第四类：权限管理资源（全局数据库 + entity 字段）

**代表资源**：Role, Permission, User, UserRole, RolePermission

**数据存储**：全局数据库表，**没有项目字段**，通过 `entity` + `entity_id` 字段关联到各层级

**隔离机制**：
- **LIST 默认不过滤**：`Role.query.all()` / `Permission.query.all()` 返回所有记录
- **查询参数可过滤**：通过 `?entity=project&entity_ids[]=uuid1,uuid2` 参数过滤
- **权限检查时匹配**：使用时通过 entity + entity_id 匹配

**代码位置**：
- `mage_ai/api/resources/RoleResource.py` [L15-L47]: collection 方法
  - 默认：`Role.query.all()`（全部返回）
  - 传了 entity 参数：按 entity + entity_ids 过滤
  - 传了 limit_roles 参数：按当前项目 access 位过滤
- `mage_ai/api/resources/PermissionResource.py` [L12-L33]: 直接使用父类 DatabaseResource.collection
- `mage_ai/api/resources/UserResource.py` [L27-L37]: 管理员过滤掉 owner

**策略层行为**：
- 策略类很简单，通常只有 `pass`
- 使用旧模式条件：`has_at_least_viewer_role()` / `has_at_least_admin_role()`
- **不使用**细粒度权限模式的 entity_name 匹配

**特点**：
- ❌ **LIST 接口默认返回全局所有数据**，没有项目隔离
- ⚠️ 依赖前端正确传 `entity` / `entity_ids` 参数来过滤
- ⚠️ 普通用户可以看到所有角色/权限的定义
- ⚠️ 权限定义本身不是机密，但用户分配关系可能敏感
- ✅ 权限**使用**时通过 entity_id 匹配，不会错用其他项目的权限

**典型 LIST 行为**：
```python
# RoleResource.collection 默认行为
roles = Role.query.all()  # 返回所有角色，不限项目

# 传了 entity=project 参数后
permissions_query = Permission.query.filter(Permission.entity == entity)
if entity != Entity.GLOBAL and entity_ids:
    permissions_query = permissions_query.filter(Permission.entity_id.in_(entity_ids))
```

---

### 5.5 第五类：工作空间与集群管理资源

**代表资源**：Workspace, Cluster, ComputeCluster 等

**数据存储**：不来自数据库，来自集群管理系统（Kubernetes API、云服务商 API 等）

**隔离机制**：
- 通过 `cluster_type` / `namespace` 等集群级概念隔离
- 项目类型校验（只有 MAIN 项目可以管理工作空间）
- `verify_project()` 方法验证子项目是否存在

**代码位置**：
- `mage_ai/api/resources/WorkspaceResource.py` [L27-L67]: collection 方法
- `mage_ai/api/resources/WorkspaceResource.py` [L158-L179]: `verify_project()` 验证

**特点**：
- ✅ 集群级隔离，物理上分开
- ⚠️ 工作空间与项目是 1:1 对应关系
- ⚠️ 只有 MAIN 项目类型可以管理工作空间

**典型验证逻辑**：
```python
def verify_project(self, subproject=None, user=None):
    project_type = get_project_type()
    if project_type != ProjectType.MAIN:
        raise ApiError('This project is ineligible for workspace management.')
    
    if project_type == ProjectType.MAIN and subproject:
        repo_path = get_repo_path(user=user)
        projects_folder = os.path.join(repo_path, 'projects')
        projects = [...]  # 扫描项目目录
        if subproject not in projects:
            raise ApiError(f'Project {subproject} was not found.')
```

---

### 5.6 第六类：运行时与系统资源

**代表资源**：Kernel, SparkApplication, SparkJob, Log, Status, Scheduler, CacheItem 等

**数据存储**：当前进程内存、本地文件系统或进程间通信

**隔离机制**：
- 进程级隔离：每个 Mage 实例只管理自己的运行时资源
- 项目切换时整个进程上下文切换

**特点**：
- ✅ 进程级天然隔离
- ⚠️ 单实例单项目模式下不存在跨项目问题
- ⚠️ 多项目部署下需要依赖进程级隔离

**策略层行为**：通常是简单的角色检查（viewer/editor/admin），不涉及细粒度的 entity 匹配

---

### 5.7 分类总表

| 类别 | 代表资源 | 数据来源 | 项目隔离方式 | 隔离可靠性 | LIST 是否默认过滤 |
|------|---------|---------|-------------|-----------|-----------------|
| 1. 文件系统类 | Pipeline, Block, File | 文件系统 | `repo_path` 目录限制 | ✅ 高 | 是 |
| 2. repo_query 类 | PipelineSchedule, Secret | 数据库 | `repo_query` (repo_path 字段) | ✅ 中高 | 是 |
| 3. 关联间接类 | PipelineRun, BlockRun | 数据库 | 通过上游 schedule/pipeline 间接过滤 | ⚠️ 中 | 是（但有绕过参数） |
| 4. 权限管理类 | Role, Permission, User | 全局数据库 | entity+entity_id 字段（使用时） | ⚠️ 中低 | **否，返回全部** |
| 5. 工作空间类 | Workspace, Cluster | 集群系统 | cluster_type / namespace | ✅ 高 | 是 |
| 6. 运行时类 | Kernel, Spark*, Status | 进程内存 | 进程级隔离 | ✅ 高 | 是 |

### 5.8 策略层与数据层的对应关系

| 类别 | 策略类定义 entity | 使用旧模式条件 | 使用新模式条件 | 缓存污染影响 |
|------|------------------|---------------|---------------|-------------|
| 1. 文件系统类 | 部分有（Pipeline 有） | 是 | 是 | 同项目内越权 |
| 2. repo_query 类 | 部分有 | 是 | 是 | 同项目内越权 |
| 3. 关联间接类 | 部分有（PipelineRun 有） | 是 | 是 | 同项目内越权 |
| 4. 权限管理类 | ❌ 没有 | ✅ 全部使用旧模式 | ❌ 不使用 | 不适用（旧模式） |
| 5. 工作空间类 | 自定义 | 自定义 | - | 不适用 |
| 6. 运行时类 | 通常没有 | 是 | 部分是 | 进程内影响 |

**关键结论**：
1. **不是所有资源都走细粒度权限路径**。权限管理类资源（Role/Permission/User）的策略类全部使用旧模式的 `has_at_least_admin_role()` 等条件，不涉及 entity_name 匹配。
2. **数据层隔离比权限层隔离更重要**。对于 1~3 类资源，即使权限系统失效，数据层过滤也能防止跨项目数据泄漏。
3. **第 4 类资源（权限管理）是特殊的**：LIST 默认返回全局数据，但这些数据是"权限定义"而非"业务数据"，泄漏风险不同。

---

## 六、多租户隔离机制对比

### 6.1 旧模式下的多租户

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

### 6.2 新模式下的多租户

新模式**不使用 `Entity` 枚举层级**（GLOBAL/PROJECT/PIPELINE），理论上通过 `entity_name + entity_id` 实现细粒度控制，但实际有三道防线共同作用于跨项目隔离。注意：这里的分析主要针对**第 1~3 类资源**（文件系统类、repo_query 类、关联间接类），第 4~6 类资源的隔离方式不同，详见第五章。

| 防线层级 | 实现方式 | 可靠性 | 代码位置 |
|----------|---------|--------|---------|
| 数据查询层 | `repo_path` 过滤 | ✅ 高 | `mage_ai/api/resources/PipelineResource.py` [L135-L204] |
| 权限系统层 | `entity_name + entity_id` 匹配 | ⚠️ 中（有缺陷） | `mage_ai/api/policies/mixins/user_permissions.py` [L117-L127] |
| 缓存层 | 请求级 ResultSet | ✅ 高（不会跨请求） | `mage_ai/api/mixins/result_set.py` [L81-L170] |

#### 6.2.1 数据查询层的项目隔离（最可靠防线）

**核心机制**：所有数据查询通过 `repo_path` 限制在当前项目范围内。

**代码位置**：
- `mage_ai/api/resources/PipelineResource.py` [L204]: `Pipeline.get_all_pipelines(repo_path=repo_path)`
- `mage_ai/settings/repo.py` [L34-L64]: `get_repo_path()` 根据请求上下文确定当前项目
- `mage_ai/server/server.py` [L784]: 服务器启动时 `set_repo_path(project)` 初始化项目路径

**查询流程**：
```
LIST /pipelines
    ↓
PipelineResource.collection()
    ├─ repo_path = get_repo_path()  ← 获取当前项目路径
    └─ Pipeline.get_all_pipelines(repo_path=repo_path)  ← 仅查询当前项目数据
        ↓ （数据层面已过滤，不会返回其他项目的 pipeline）
    返回当前项目的 pipeline 列表
```

**关键结论**：即使权限系统有缺陷，**数据查询层天然保证了不会返回其他项目的数据**。

#### 6.2.2 权限系统层的四种场景分析

需要严格区分以下四种不同场景，它们的影响范围和后果完全不同：

##### 场景 1：列表操作级放行

**触发条件**：
- LIST 操作的阶段 1 检查（操作级）
- `policy = PipelinePolicy(None, user)` → resource=None
- `permission.entity_id is not None and resource` → False（AND 逻辑）

**代码位置**：`mage_ai/api/policies/mixins/user_permissions.py` [L117-L127]
```python
if permission.entity_id is not None and resource:  # 两个条件同时满足才检查
```

**影响范围**：
- ✅ 仅影响"能不能调用 LIST 接口"
- ❌ 不影响数据返回范围（数据查询层已过滤）
- ⚠️ 带具体 entity_id 的权限被当作全量权限使用

**实际后果**：
- 用户只有项目 A 中 pipeline-a 的权限 → 在项目 B 中调用 LIST /pipelines
- 阶段 1 操作级检查：✅ 通过（entity_id 检查被跳过）
- 阶段 3 数据查询：仅返回项目 B 的 pipeline（repo_path 过滤）
- 阶段 4 属性级检查：逐个验证项目 B 的 pipeline
- **结果**：不会泄漏项目 A 的数据，只是可能放行本应拒绝的接口调用

##### 场景 2：列表属性级缓存污染

**触发条件**：
- LIST 操作的阶段 4 检查（属性级，逐个资源）
- 同一 ResultSet 内多个同类型资源共享缓存
- 缓存 key：`entity_name → operation → authorized`（不含 entity_id）

**代码位置**：`mage_ai/api/mixins/result_set.py` [L81-L170]

**缓存作用域**：
- ✅ 请求级别：每个请求创建独立的 ResultSet（`BasePolicy.__init__` [L62]）
- ✅ 不跨请求：不同请求的 ResultSet 互不影响
- ⚠️ 同一请求内：所有同类型资源共享缓存

**影响范围**：
- ⚠️ 仅限于**当前请求**返回的**当前项目**数据范围内
- ❌ 不会跨项目（数据查询层已过滤）
- ❌ 不会跨请求（ResultSet 是请求级的）

**实际后果**：
- 当前项目返回 [pipeline-1, pipeline-2, pipeline-3]
- 用户只有 pipeline-1 的权限
- pipeline-1 检查通过 → 缓存 `operations → Pipeline → LIST → True`
- pipeline-2 和 pipeline-3 查缓存 → 全部通过
- **结果**：用户可以看到当前项目中原本无权限的资源，但**不会看到其他项目的数据**

##### 场景 3：创建操作放行

**触发条件**：
- CREATE 操作的阶段 1 和阶段 2 检查
- resource 始终为 None（资源还未创建）
- entity_id 检查始终被跳过

**代码位置**：`mage_ai/api/operations/base.py` [L500-L529]

**影响范围**：
- ✅ 影响"能不能创建资源"
- ✅ 创建的资源存储在当前项目 repo_path 下
- ❌ 不会跨项目创建

**实际后果**：
- 用户只有项目 A 中特定 pipeline 的编辑权限
- 用户在项目 B 中调用 POST /pipelines 创建新 pipeline
- 操作级检查：✅ 通过（resource=None，entity_id 检查跳过）
- 写属性检查：✅ 通过（resource=None）
- 实际创建：新 pipeline 存储在项目 B 的 repo_path 下
- **结果**：权限粒度控制失效，但不会跨项目创建资源

##### 场景 4：真实跨项目泄漏

**真实跨项目泄漏需要同时满足以下所有条件**：

| 条件 | 说明 | 默认状态 |
|------|------|---------|
| 1. 权限加载不按项目过滤 | `load_and_cache_user_permissions()` 查询时只有 `user_id` 过滤，没有项目过滤 | ❌ 默认满足 |
| 2. 权限验证不使用 Entity 层级 | 新模式完全不检查 `Permission.entity` 字段 | ❌ 默认满足 |
| 3. 数据查询层不按 repo_path 过滤 | `get_all_pipelines()` 等方法没有使用 repo_path 参数 | ✅ 默认不满足（数据层有过滤） |
| 4. 缓存跨请求共享 | ResultSet 在多个请求间共享 | ✅ 默认不满足（ResultSet 是请求级的） |

**结论**：
- ⚠️ 权限系统本身存在设计缺陷（条件 1、2 默认满足）
- ✅ 但数据查询层的 `repo_path` 过滤和 ResultSet 的请求级作用域提供了有效的兜底保护
- ❌ **在默认配置下，不会发生真实的跨项目数据泄漏**
- ⚠️ 但权限粒度控制失效（同项目内越权访问）是真实存在的问题

#### 6.2.3 权限系统的实际问题（非跨项目）

虽然不会跨项目泄漏，但新模式在**同项目内**存在以下权限控制缺陷：

| 问题 | 影响 | 条件 |
|------|------|------|
| LIST 接口过度放行 | 本应拒绝的用户可以调用列表接口 | resource=None 时 entity_id 检查跳过 |
| 同项目内越权查看 | 用户可以看到项目内原本无权限的资源 | 缓存污染（同请求内） |
| CREATE 权限过度放行 | 本应只能编辑特定资源的用户可以创建新资源 | CREATE 时 resource 始终为 None |
| 禁用权限范围过大 | 一个禁用权限可能影响整个资源类型 | 不检查 entity 层级 + resource=None 时跳过检查 |

#### 6.2.4 易混淆点澄清

| 说法 | 准确性 | 说明 |
|------|--------|------|
| "缓存污染会跨项目" | ❌ 错误 | 数据查询层已按项目过滤，缓存只作用于当前项目数据 |
| "权限系统缺陷会导致跨项目泄漏" | ❌ 错误 | 数据层的 repo_path 过滤是独立且可靠的防线 |
| "新模式完全没有项目隔离" | ❌ 错误 | 数据层有 repo_path 隔离，只是权限层不感知项目 |
| "entity_id 精确匹配完全失效" | ❌ 错误 | 在 DETAIL/UPDATE/DELETE 等单资源操作中仍然有效 |
| "缓存会跨请求泄漏" | ❌ 错误 | ResultSet 是请求级的，每个请求独立 |

#### 6.2.5 重要提醒

新模式的权限系统存在设计缺陷，但**不会导致跨项目数据泄漏**。实际风险是：
1. **同项目内的权限粒度控制失效**（用户可以看到项目内原本无权限的资源）
2. **LIST/CREATE 接口的操作级权限检查失效**（过度放行）
3. **禁用权限的影响范围可能超出预期**（跨资源类型生效）

如果需要严格的细粒度权限控制，需要修复以下问题：
1. resource=None 时的 entity_id 检查逻辑
2. 缓存 key 中加入 entity_id 维度
3. 权限加载时增加项目级过滤

---

## 七、失败分支详细对比

| 失败类型 | 旧模式 | 新模式 | 共同 |
|----------|--------|--------|------|
| 无 action 规则配置 | ✅ | ✅（自动生成规则，不会出现） | ✅ |
| Scope 验证失败 | ✅ | ✅ | ✅ |
| 条件函数返回 False | ✅ (has_at_least_... 等) | ✅ (validate_condition_...) | 机制相同，函数不同 |
| Entity 层级不匹配 | ✅（有继承兜底） | ❌（不使用 Entity 层级） | - |
| entity_name 不匹配 | ❌ | ✅ | - |
| entity_id 不匹配（resource 存在时） | ✅（但继承可绕过） | ✅（无继承，精确匹配） | 机制不同 |
| entity_id 检查被跳过（resource 为空时） | ❌ | ✅（LIST/CREATE 操作级） | - |
| 禁用权限优先 | ❌（位或聚合，禁用位不会抵消授权位） | ✅（显式禁用优先逻辑） | - |
| 属性级禁用 | ❌（属性级只有 allow/deny） | ✅（disabled_attributes 白/黑名单） | - |
| 缓存导致隐性绕过（同请求内） | ❌ | ✅（缓存 key 不含 entity_id） | - |
| 禁用权限跨资源类型影响 | ❌（项目层级隔离） | ⚠️（可能，但不跨项目数据） | - |
| LIST 整体 403 | ❌（按项目过滤结果） | ✅（一个不通过全部失败） | - |
| 跨项目数据泄漏 | ❌（有 Entity.PROJECT 隔离） | ❌（数据层 repo_path 隔离兜底） | ✅ 都不会 |
| DEBUG 调试信息 | ✅ | ✅ | ✅ |
| 指标埋点 | ✅ | ✅ | ✅ |
| 无项目权限友好提示 | ✅（旧模式特有） | ❌ | - |

### 7.1 旧模式失败流程图

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
           [BaseOperation.execute]
           ├─ 403 + 无项目权限 → 修改错误消息
           ├─ 运行失败钩子
           └─ DEBUG 模式重新抛出
```

### 7.2 新模式失败流程图（完整链路）

```
validate_condition_with_cache()  ← 入口
    │
    ├─ 查缓存
    │   ├─ 命中 → 直接返回结果（跳过所有检查！）
    │   └─ 未命中 → 继续执行
    │
    └─ validate_condition_with_permissions()
        │
        ├─ 全局 Owner 检查：任一 permission 含 OWNER → ✅ 通过
        │
        └─ 并行检查所有 permission
            │
            ├─ permission.access 为空？→ 跳过 (False, False)
            ├─ entity_name 不匹配？→ 跳过 (False, False)
            │
            ├─ resource 存在 且 permission.entity_id 有值？
            │   ├─ 是 → 比较 entity_id → 不匹配则跳过 (False, False)
            │   └─ 否 → entity_id 检查被跳过 → 继续
            │
            ├─ disable_access 匹配？→ ❌ 禁用 (False, True)
            ├─ 属性操作全禁用？→ ❌ 禁用 (False, True)
            ├─ 具体属性被禁用？→ ❌ 禁用 (False, True)
            │
            ├─ ALL 权限？→ ✅ 授权 (True, False)
            ├─ OPERATION_ALL + 具体操作？→ ✅ 授权 (True, False)
            ├─ 具体操作位 + 属性位？→ ✅ 授权 (True, False)
            │
            └─ 不匹配 → 跳过 (False, False)
        │
        └─ 结果聚合：
            authorized = any(permission_granted)     ← 任一授权则 True
            unauthorized = any(permission_disabled)   ← 任一禁用则 True
            return authorized and not unauthorized    ← 禁用优先
    │
    └─ 写入缓存（无论成功失败都缓存）
```

### 7.3 LIST 操作特殊失败路径

```
GET /pipelines (LIST)
    │
    ├─ 阶段 1：操作级检查（resource=None）
    │   ├─ entity_id 检查被跳过
    │   ├─ 任一权限授权 → ✅ 通过（即使只授权了某个 entity_id）
    │   └─ 全部不授权或被禁用 → ❌ 403
    │
    ├─ 阶段 2：查询级检查（resource=None）
    │   └─ 同操作级
    │
    ├─ 阶段 3：获取数据列表
    │   └─ 返回 N 个资源
    │
    └─ 阶段 4：属性级检查（逐个 resource）
        │
        ├─ 资源 1：entity_id 匹配 → ✅ 授权 → 写入缓存 True
        │
        ├─ 资源 2：entity_id 不匹配
        │   ├─ 查缓存 → 命中 True ← 缓存污染！
        │   └─ 直接返回 True ⚠️（本应失败）
        │
        ├─ ... 后续资源全部因缓存污染通过
        │
        └─ 或者（如果第一个资源就不匹配）：
            ├─ 资源 1：entity_id 不匹配 → ❌ 不授权
            ├─ 所有资源都不授权 → ❌ 403
            └─ 整个请求失败，而不是过滤掉无权限资源
```

---

## 八、关键代码位置索引

### 8.1 旧模式（角色权限路径）

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

### 8.2 新模式（细粒度权限路径）

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

### 8.3 共用框架

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

### 8.4 资源与数据层隔离

| 功能 | 文件相对路径 | 行号 |
|------|-------------|------|
| PipelineResource.collection | `mage_ai/api/resources/PipelineResource.py` | L104-L204 |
| RoleResource.collection | `mage_ai/api/resources/RoleResource.py` | L15-L63 |
| PermissionResource.collection | `mage_ai/api/resources/PermissionResource.py` | L12-L33 |
| PipelineRunResource.collection | `mage_ai/api/resources/PipelineRunResource.py` | L38-L136 |
| WorkspaceResource.collection | `mage_ai/api/resources/WorkspaceResource.py` | L27-L67 |
| PipelineSchedule.repo_query | `mage_ai/orchestration/db/models/schedules.py` | L112-L121 |
| Secret.repo_query | `mage_ai/orchestration/db/models/secrets.py` | L19-L26 |
| get_repo_path | `mage_ai/settings/repo.py` | L34-L64 |

---

## 九、易混淆点总结

### 9.1 核心概念区分

1. **Entity vs entity_name**:
   - `Entity` (GLOBAL/PROJECT/PIPELINE): 旧模式的**层级范围**概念，有纵向继承
   - `entity_name` (Pipeline/Block/...): 新模式的**资源类型**概念，有横向通配符

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

### 9.2 资源对象为空时的特殊行为

6. **resource=None 时 entity_id 检查被跳过**:
   - 触发场景：LIST/CREATE 的操作级检查、CREATE 的写属性检查
   - 代码位置：`mage_ai/api/policies/mixins/user_permissions.py` [L117-L127]
   - 关键条件：`if permission.entity_id is not None and resource:`（AND 逻辑）
   - 后果：带具体 entity_id 的权限在列表/创建时被当作全量权限使用

7. **LIST 操作的两阶段权限检查**:
   - 阶段 1（操作级）：resource=None → entity_id 检查跳过 → 决定"能不能访问列表接口"
   - 阶段 2（属性级）：逐个 resource 检查 → entity_id 检查生效 → 决定"能不能读具体资源的字段"
   - 注意：阶段 2 中任何一个资源失败 → 整个请求 403，而非过滤掉无权限资源

### 9.3 缓存相关的陷阱

8. **缓存 key 不含 entity_id**:
   - 操作级缓存 key：`entity_name → operation → authorized`
   - 属性级缓存 key：`entity_name → operation → attribute_operation → attribute → authorized`
   - 后果：同类型不同资源共享缓存，第一个资源的结果决定全部

9. **缓存污染的范围（重要纠正）**:
   - ✅ 仅在**同一请求**内的**同一 ResultSet** 中生效
   - ❌ 不会跨请求（每个请求新建独立的 ResultSet）
   - ❌ 不会跨项目（数据查询层已按 repo_path 过滤）
   - ⚠️ 同一请求内：只要有一个资源授权通过 → 同类型所有资源都被认为有权限
   - ⚠️ 同一请求内：只要有一个资源被禁用 → 同类型所有资源都被认为被禁用
   - 遍历顺序决定最终结果，具有不确定性

10. **缓存的作用域**:
    - 存储位置：`ResultSet.context.data`
    - 共享范围：同一个 ResultSet 中的所有资源共享
    - 跨请求：不共享（每次请求新建 ResultSet）
    - 初始化位置：`BasePolicy.__init__` [L62] - resource=None 时创建空 ResultSet

### 9.4 禁用优先原则的影响

11. **禁用优先 vs 位或聚合**:
    - 旧模式：位或运算，禁用位不会抵消授权位（没有显式禁用概念）
    - 新模式：`authorized and not unauthorized`，任一禁用则整体拒绝

12. **禁用权限的影响范围（重要纠正）**:
    - ⚠️ 新模式不检查 Permission.entity 层级，禁用权限可能跨**资源类型**生效
    - ⚠️ resource=None 时，entity_id 检查被跳过，禁用影响范围更大
    - ❌ 但**不会跨项目泄漏数据**（数据层 repo_path 过滤是独立防线）
    - ❌ 不会导致其他项目的数据被返回，只是可能错误地拒绝当前项目的访问

### 9.5 多租户隔离效果总结

13. **新模式下的实际隔离能力（重要纠正）**:
    | 操作类型 | 权限粒度控制 | 跨项目数据隔离 | 说明 |
    |----------|-------------|---------------|------|
    | DETAIL/UPDATE/DELETE | ✅ 基本有效 | ✅ 有效 | 单资源操作，entity_id 精确匹配 + 数据层过滤 |
    | LIST 操作级 | ❌ 完全失效 | ✅ 有效 | resource=None 时 entity_id 检查跳过，但数据层仍过滤 |
    | LIST 属性级 | ❌ 基本失效 | ✅ 有效 | 缓存污染（同请求内），但数据层已过滤 |
    | CREATE | ❌ 完全失效 | ✅ 有效 | resource 始终为 None，但创建的数据在当前项目 |
    | 跨项目数据泄漏 | - | ✅ 有效 | 数据层 repo_path 过滤提供可靠兜底 |

14. **新旧模式对比**:
    - 旧模式：粗粒度但可靠的项目级隔离（Entity.PROJECT 层级）
    - 新模式：细粒度权限控制有缺陷，但数据层隔离兜底
    - 注意：新模式的 entity_id 精确匹配在单资源操作时是可靠的，但列表和创建场景存在设计缺陷
    - 关键区别：旧模式在**权限层**实现项目隔离，新模式在**数据层**实现项目隔离

### 9.6 跨项目影响的澄清

15. **不会导致跨项目数据泄漏的三道防线**:
    - 防线 1：数据查询层 `repo_path` 过滤 → 最可靠，默认启用
    - 防线 2：ResultSet 请求级作用域 → 缓存不会跨请求
    - 防线 3：项目切换时重新设置 repo_path → 服务器级别的项目上下文

16. **权限系统的真实风险（非跨项目）**:
    - 同项目内越权访问（用户可以看到项目内原本无权限的资源）
    - LIST/CREATE 接口过度放行（本应拒绝的用户可以调用接口）
    - 禁用权限影响范围过大（可能错误地拒绝当前项目的合法访问）
    - 权限粒度控制失效（entity_id 精确匹配在列表场景无效）

17. **真实跨项目泄漏的必要条件（全部满足才会发生）**:
    - 条件 1：权限加载不按项目过滤（默认满足）
    - 条件 2：权限验证不使用 Entity 层级（默认满足）
    - 条件 3：**数据查询层不按 repo_path 过滤**（默认不满足，需要代码缺陷）
    - 条件 4：**缓存跨请求共享**（默认不满足，ResultSet 是请求级的）
    - 结论：默认配置下不会发生跨项目数据泄漏

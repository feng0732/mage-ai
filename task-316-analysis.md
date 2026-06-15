# Mage AI Secrets 管理模块深度分析

## 一、整体架构概览

Mage AI 的 Secrets 管理采用**分层架构**设计，共分为 6 个核心层次，从上到下依次为：

```
┌─────────────────────────────────────────────────────┐
│  API 层 (SecretResource)                            │
│  提供 RESTful 接口，处理 HTTP 请求/响应              │
├─────────────────────────────────────────────────────┤
│  业务逻辑层 (shared/secrets.py)                     │
│  Secret CRUD、加密解密、多实体隔离                   │
├─────────────────────────────────────────────────────┤
│  数据模型层 (models/secrets.py)                     │
│  Secret ORM 模型、数据库持久化                       │
├─────────────────────────────────────────────────────┤
│  配置后端层 (settings/backends.py)                  │
│  SettingsBackend 抽象 + AWS Secrets Manager 实现    │
├─────────────────────────────────────────────────────┤
│  IO 配置层 (io/config.py)                           │
│  AWSSecretLoader / EnvironmentVariableLoader        │
├─────────────────────────────────────────────────────┤
│  数据库基础层 (db/__init__.py, models/base.py)      │
│  safe_db_query 装饰器、事务回滚、重试机制            │
└─────────────────────────────────────────────────────┘
```

---

## 二、模块职责详解

### 2.1 数据模型层 - [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/secrets.py)

**核心职责**：定义 Secret 的数据库存储结构和查询接口。

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | String(255) | Secret 名称，唯一约束的一部分 |
| `value` | Text | **加密后**的 Secret 值，不存储明文 |
| `repo_name` | String(255) | 仓库路径标识，用于多仓库/多项目隔离 |
| `key_uuid` | String(255) | 加密密钥的 UUID，关联密钥文件 |

**关键设计**：
- 唯一约束：`UniqueConstraint('name', 'key_uuid')` — 同一密钥域下名称唯一
- [repo_query](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/secrets.py#L19-L26) 类属性：自动过滤当前仓库或全局的 secrets
- [get_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/secrets.py#L28-L37) 方法：支持 `key_uuid` 带空白字符的容错查询

```python
@classmethod
@safe_db_query
def get_secret(cls, name: str, key_uuid: str) -> Optional['Secret']:
    return cls.query.filter(
        cls.name == name,
        or_(
            cls.key_uuid == key_uuid,
            cls.key_uuid == key_uuid.strip(),  # 容错：处理 UUID 文件中的空白
        ),
    ).one_or_none()
```

### 2.2 业务逻辑层 - [shared/secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py)

这是 Secrets 管理的**核心大脑**，负责所有业务逻辑。

#### 2.2.1 多实体密钥目录体系

通过 [Entity](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/constants.py#L21-L27) 枚举实现三级隔离：

| Entity 级别 | 目录结构 | 适用场景 |
|-------------|---------|---------|
| `GLOBAL` | `{data_dir}/secrets/` | 全局共享密钥 |
| `PROJECT` | `{data_dir}/secrets/projects/{project_uuid}/` | 项目级密钥 |
| `PIPELINE` | `{data_dir}/secrets/projects/{project_uuid}/pipelines/{pipeline_uuid}/` | 流水线级密钥 |

[get_secrets_dir](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L16-L53) 根据实体类型动态计算密钥存储目录。

#### 2.2.2 密钥文件管理

每个实体目录下包含两个文件：
- `key`：Fernet 对称加密密钥（二进制格式）
- `uuid`：该密钥的唯一标识 UUID

[_create_key_files](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L247-L280) 负责密钥的**懒创建**：
- 目录不存在时自动 `os.makedirs`
- `key` 文件不存在时用 `Fernet.generate_key()` 生成新密钥
- `uuid` 文件不存在时用 `uuid.uuid4().hex` 生成新 UUID

#### 2.2.3 核心 API 函数

| 函数 | 职责 | 关键状态 |
|------|------|---------|
| [create_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L87-L126) | 创建并加密存储 Secret | 校验参数 → 获取/创建密钥 → Fernet 加密 → 持久化 |
| [get_secret_value](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L158-L202) | 解密获取 Secret 明文值 | 读取密钥 → 查询 DB → Fernet 解密 → 兼容旧数据 |
| [delete_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L215-L244) | 删除 Secret 记录 | 查询定位 → DB 删除 |
| [get_valid_secrets_for_repo](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L129-L155) | 过滤当前仓库可用的 secrets | 逐一尝试解密，跳过 InvalidToken 的记录 |

#### 2.2.4 向后兼容机制

在 [get_secret_value](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L185-L199) 中实现了双轨查询：

1. **新逻辑（优先）**：用 `key_uuid` 精确匹配 Secret
2. **旧逻辑（回退）**：若未找到，且是 GLOBAL 实体，则查询 `key_uuid IS NULL` 的旧格式记录

### 2.3 API 层 - [SecretResource.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py)

**职责**：将业务逻辑暴露为 REST API。

| HTTP 方法 | 端点 | 对应方法 | 说明 |
|-----------|------|---------|------|
| GET | `/api/secrets` | [collection](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py#L32-L46) | 列出 secrets（带权限过滤） |
| POST | `/api/secrets` | [create](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py#L48-L54) | 创建新 secret |
| GET | `/api/secrets/{name}` | [member](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py#L56-L65) | 获取单个 secret |

[_filter_secrets](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py#L67-L79) 实现了 Git 相关密钥的权限隔离：用户只能看到自己创建的 Git SSH Key / Access Token。

### 2.4 配置后端层 - [backends.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py)

**职责**：为 Mage 系统配置提供可插拔的 Secret 读取后端。

#### 类层次结构：

```
SettingsBackend (抽象基类)
└── AWSSecretsManagerBackend (AWS 实现)
```

[SettingsBackend.get()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L26-L42) 的三级回退策略：
```
优先级 1: 调用 _get() 从具体后端（如 AWS）读取
优先级 2: os.getenv() 从环境变量读取
优先级 3: kwargs['default'] 返回默认值
```

[AWSSecretsManagerBackend](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L58-L96) 特性：
- 支持可选的 `prefix` 配置，为所有 secret 名称加前缀
- 可选的 `aws_secretsmanager_caching` 本地缓存
- 对 `ResourceNotFoundException` 静默返回 None，其他错误记录日志

### 2.5 IO 配置层 - [io/config.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/io/config.py)

**职责**：为 IO 连接器（数据库、数据仓库等）提供配置加载能力。

| 加载器类 | 配置源 | 用途 |
|---------|--------|------|
| `AWSSecretLoader` | AWS Secrets Manager | 从 AWS 加载连接器凭据 |
| `EnvironmentVariableLoader` | 操作系统环境变量 | 从环境变量读取凭据 |
| `ConfigKey` 枚举 | - | 所有支持的配置项常量列表（AWS、Postgres、Snowflake 等 50+ 项） |

### 2.6 数据库基础层

#### 2.6.1 safe_db_query 装饰器 - [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171)

这是整个数据访问层的**安全网**，为所有数据库操作提供：

```python
def safe_db_query(func):
    def func_with_rollback(*args, **kwargs):
        retry_count = 0
        while True:
            try:
                return func(*args, **kwargs)
            except (
                sqlalchemy.exc.OperationalError,      # 连接断开等操作错误
                sqlalchemy.exc.PendingRollbackError,   # 事务需要回滚
                sqlalchemy.exc.InternalError,          # DB 内部错误
            ) as e:
                db_connection.session.rollback()        # 关键：事务回滚
                if retry_count >= DB_RETRY_COUNT:       # DB_RETRY_COUNT = 2
                    raise e
                retry_count += 1
    return func_with_rollback
```

#### 2.6.2 BaseModel 事务保护 - [base.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py)

[save()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L67-L81)、[update()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L83-L93)、[delete()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L95-L102) 三个写操作均内置事务回滚：

```python
def save(self, commit=True) -> None:
    # ... 字段校验 ...
    self.session.add(self)
    if commit:
        try:
            self.session.commit()
        except Exception as e:
            self.session.rollback()  # 失败立即回滚
            raise e
```

---

## 三、关键状态变化与生命周期

### 3.1 Secret 创建生命周期

以调用 `create_secret('DB_PASSWORD', 'my_secret_123')` 为例：

```
阶段 1: 参数校验
    │
    ├─ entity=PROJECT/PIPELINE 时检查 project_uuid 是否缺失
    ├─ entity=PIPELINE 时检查 pipeline_uuid 是否缺失
    └─ 缺失 → 抛出 Exception("Missing values for creating secret...")
    │
阶段 2: 密钥准备
    │
    ├─ 调用 get_secrets_dir() 确定密钥目录路径
    ├─ 调用 _create_key_files(secrets_dir)
    │   ├─ 目录不存在 → os.makedirs 创建
    │   ├─ key 文件不存在 → Fernet.generate_key() 生成并写入
    │   └─ uuid 文件不存在 → uuid.uuid4().hex 生成并写入
    └─ 返回 (key: str, key_uuid: str)
    │
阶段 3: 加密
    │
    ├─ Fernet(key) 初始化加密器
    ├─ value.encode('utf-8') → 明文转字节
    └─ fernet.encrypt(plaintext_bytes) → 加密字节转 UTF-8 字符串
    │
阶段 4: 持久化
    │
    ├─ 构造 Secret(name, encrypted_value, key_uuid, repo_name)
    ├─ secret.save()
    │   ├─ session.add(secret)
    │   └─ session.commit()（失败自动 rollback）
    └─ 返回 Secret 实例
```

### 3.2 Secret 读取生命周期

以 `get_secret_value('DB_PASSWORD')` 为例：

```
阶段 1: 获取加密密钥
    │
    ├─ 读取 {secrets_dir}/key 文件 → key
    │   └─ 文件不存在/读取异常 → key = None
    ├─ 读取 {secrets_dir}/uuid 文件 → key_uuid
    │   └─ 文件不存在/读取异常 → key_uuid = None
    └─ key 为空 → 输出 WARNING → 返回 None
    │
阶段 2: 查询新格式记录
    │
    ├─ Secret.get_secret(name, key_uuid)
    └─ 找到 → 进入解密阶段
    │
阶段 3: 解密（带错误容错）
    │
    ├─ fernet.decrypt(secret.value.encode('utf-8'))
    ├─ 成功 → 返回 .decode('utf-8') 明文字符串
    └─ 失败 InvalidToken → 静默跳过，继续尝试
    │
阶段 4: 向后兼容回退（仅 GLOBAL 实体）
    │
    ├─ 查询 Secret(name=xxx, repo_name=xxx, key_uuid IS NULL)
    ├─ 找到 → 尝试解密
    │   ├─ 成功 → 返回明文
    │   └─ 失败 InvalidToken → 跳过
    └─ 均未找到 → 输出 WARNING → 返回 None
```

### 3.3 密钥文件状态转换

```
            目录不存在
                │
                ▼
        os.makedirs() 创建
                │
                ▼
         ┌─────────────┐
         │  目录已创建   │
         └─────────────┘
          │            │
          ▼            ▼
    key 文件?      uuid 文件?
     存在?          存在?
     │   │          │    │
    是   否         是    否
    │    │         │     │
    │    ▼         │     ▼
    │ Fernet.      │ uuid.uuid4()
    │ generate_key()│    .hex
    │    │         │     │
    │    ▼         │     ▼
    │ 写入文件    │   写入文件
    └────┘         └─────┘
         │
         ▼
   密钥对可用状态
```

---

## 四、失败回退机制详解

### 4.1 数据库层面的回退（三层防护）

| 层级 | 机制 | 触发条件 | 回退动作 |
|------|------|---------|---------|
| **L1: BaseModel** | `save/update/delete` 内置 try-catch | 任何 commit 异常 | `session.rollback()`，异常上抛 |
| **L2: safe_db_query** | 装饰器重试 + 回滚 | OperationalError / PendingRollbackError / InternalError | 先 rollback，最多重试 2 次，仍失败则上抛 |
| **L3: DBConnection** | 连接级管理 | session 非活跃时自动重建 | `close_session()` → `start_session()` |

**回退调用链示例**（创建 Secret 失败场景）：

```
create_secret()
  └─ secret.save()
       ├─ session.add(secret)
       └─ try: session.commit()
          └─ 失败（如唯一约束冲突）
              ├─ session.rollback()   ← L1 回退
              └─ raise e
              └─ (如为 OperationalError 类)
                  └─ safe_db_query 捕获
                      ├─ session.rollback()  ← L2 再次回退（幂等）
                      ├─ retry_count++
                      └─ 重试 < 2 次 → 重新执行原函数
```

### 4.2 加密解密层面的回退

#### 4.2.1 InvalidToken 容错

在 [get_secret_value](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L180-L183) 和 [get_valid_secrets_for_repo](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L148-L153) 中：

```python
try:
    return fernet.decrypt(secret.value.encode('utf-8')).decode('utf-8')
except InvalidToken:
    pass  # 不抛出异常，静默跳过该条记录
```

**设计意图**：
- 当密钥轮换后，旧密钥加密的数据无法解密时，系统不会崩溃
- `get_valid_secrets_for_repo` 会自动过滤掉无法解密的"脏数据"

#### 4.2.2 密钥文件读取容错

[_get_encryption_key](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L296-L306) 对两个密钥文件的读取异常完全屏蔽：

```python
try:
    with open(key_file, 'r', encoding='utf-8') as f:
        key = f.read()
except Exception:
    key = None  # 任何异常（文件不存在、权限不足、损坏等）都视为无密钥
```

### 4.3 后端配置层面的回退

[SettingsBackend.get()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L26-L42) 三级回退：

```
调用 settings.get_value('AWS_ACCESS_KEY_ID')
    │
    ├─ 第 1 级: AWSSecretsManagerBackend._get() → AWS API
    │   └─ 失败/None → 继续
    │
    ├─ 第 2 级: os.getenv('AWS_ACCESS_KEY_ID') → 环境变量
    │   └─ 不存在 → 继续
    │
    └─ 第 3 级: kwargs.get('default') → 默认值
        └─ 无默认值 → 返回 None
```

### 4.4 API 层面的回退

[SecretResource.member()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py#L56-L65)：

```python
model = Secret.query.filter(Secret.repo_name == repo_path, Secret.name == pk).first()
if not model:
    raise ApiError(ApiError.RESOURCE_NOT_FOUND)  # 返回 404
```

### 4.5 管道重命名时的目录迁移回退

[rename_pipeline_secrets_dir()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L56-L74)：

```python
if os.path.exists(secrets_dir):
    shutil.move(old_path, new_path)  # 原子性移动，不存在不操作
```

**安全特性**：源目录不存在时跳过，不抛异常，天然具有幂等性。

---

## 五、关键代码路径索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Secret 数据模型 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/secrets.py#L10-L37) | L10-L37 |
| 创建 Secret | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L87-L126) | L87-L126 |
| 读取 Secret | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L158-L202) | L158-L202 |
| 删除 Secret | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L215-L244) | L215-L244 |
| 密钥文件创建 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L247-L280) | L247-L280 |
| 数据库安全装饰器 | [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171) | L155-L171 |
| BaseModel 事务回滚 | [base.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L67-L102) | L67-L102 |
| AWS 配置后端 | [backends.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L58-L96) | L58-L96 |
| Settings 单例 | [settings/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/__init__.py#L9-L49) | L9-L49 |
| Secret REST API | [SecretResource.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/api/resources/SecretResource.py#L29-L79) | L29-L79 |
| 单元测试 | [test_secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/tests/data_preparation/storage/shared/test_secrets.py#L24-L166) | L24-L166 |
| API 测试 | [test_secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/tests/api/endpoints/test_secrets.py#L15-L62) | L15-L62 |

---

## 六、设计亮点总结

1. **三级实体隔离**：GLOBAL / PROJECT / PIPELINE，通过文件目录和 key_uuid 双重隔离
2. **分层防御式回退**：从 BaseModel → safe_db_query → 业务逻辑层 → API 层，每层都有错误处理
3. **Fernet 对称加密**：行业标准加密库，密钥与密文分离存储（密钥在文件系统，密文在数据库）
4. **向后兼容双轨查询**：支持 key_uuid 新旧两种格式的无缝迁移
5. **解密容错**：InvalidToken 异常不中断流程，自动跳过损坏数据
6. **可插拔配置后端**：SettingsBackend 支持替换为 AWS Secrets Manager 等外部服务
7. **幂等操作设计**：密钥文件懒创建、目录删除前检查存在性，保证重复调用安全

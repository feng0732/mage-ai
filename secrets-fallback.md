# Secrets 管理失败回退边界深度剖析

## 核心结论速览

Secrets 管理的回退机制分为**四个独立层级**，每层的触发条件、作用范围和行为边界完全不同：

| 回退类型 | 所属层级 | 核心机制 | 是否重试 | 是否静默 | 作用边界 |
|---------|---------|---------|---------|---------|---------|
| 数据库重试 | 数据访问层 | `@safe_db_query` 装饰器 | 是（2次） | 否 | 仅 3 种 SQLAlchemy 异常 |
| 事务回滚 | 数据访问层 | `session.rollback()` | 否 | 否 | 数据库会话内的操作 |
| 加密解密容错 | 业务逻辑层 | try-catch `InvalidToken` | 否 | 是 | 仅解密失败场景 |
| 配置后端降级 | 配置服务层 | 多源依次尝试 | 否 | 是 | 配置读取的整条链路 |

---

## 一、数据库重试机制边界

### 1.1 重试装饰器全貌

数据库重试图由 [safe_db_query](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171) 装饰器实现，是整个数据访问层的"安全网"。

```python
DB_RETRY_COUNT = 2

def safe_db_query(func):
    def func_with_rollback(*args, **kwargs):
        retry_count = 0
        while True:
            try:
                return func(*args, **kwargs)
            except (
                sqlalchemy.exc.OperationalError,      # ← 可重试
                sqlalchemy.exc.PendingRollbackError,   # ← 可重试
                sqlalchemy.exc.InternalError,          # ← 可重试
            ) as e:
                db_connection.session.rollback()
                if retry_count >= DB_RETRY_COUNT:
                    raise e
                retry_count += 1
    return func_with_rollback
```

### 1.2 重试边界：哪些异常会触发重试？

**✅ 会重试的 3 种异常**：

| 异常类型 | 触发场景 | 重试的合理性 |
|---------|---------|------------|
| `OperationalError` | 数据库连接断开、网络超时、连接池耗尽 | 临时故障，重连可能恢复 |
| `PendingRollbackError` | 前一个事务失败，当前会话处于待回滚状态 | 回滚后重新开始事务即可 |
| `InternalError` | 数据库内部错误（如死锁、资源不足） | 死锁等场景重试可能成功 |

**❌ 不会重试的异常（直接抛出）**：

| 异常类型 | 触发场景示例 | 不重试的原因 |
|---------|-------------|-------------|
| `IntegrityError` | 唯一约束冲突、外键约束违反 | 业务逻辑错误，重试也失败 |
| `DataError` | 数据类型不匹配、值过长 | 数据本身有问题，重试无效 |
| `ProgrammingError` | SQL 语法错误、表不存在 | 代码 bug，需修复代码 |
| `NotSupportedError` | 数据库不支持该操作 | 能力边界，重试无意义 |

### 1.3 重试次数与状态

- **最大重试次数**：`DB_RETRY_COUNT = 2` 次（总共尝试 3 次：初始 1 次 + 重试 2 次）
- **每次重试前的动作**：先执行 `session.rollback()` 清理事务状态
- **重试耗尽后的行为**：原样抛出原始异常，上抛给调用方

### 1.4 Secrets 相关函数的重试覆盖

| 函数 | 是否受 `@safe_db_query` 保护 |
|------|---------------------------|
| [create_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L87-L126) | ✅ 是 |
| [get_secret_value](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L158-L202) | ✅ 是 |
| [delete_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L215-L244) | ✅ 是 |
| [get_valid_secrets_for_repo](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L129-L155) | ❌ 否（无装饰器） |
| `_create_key_files` | ❌ 否（纯文件操作，不涉及 DB） |
| `_get_encryption_key` | ❌ 否（纯文件操作） |

> **重要边界**：`_create_key_files` 在 `create_secret` 内部调用，位于装饰器包裹范围内？**不**。`_create_key_files` 是文件系统操作，不在数据库事务中，即使后续 DB 操作失败回滚，已创建的密钥文件**不会被删除**。

---

## 二、事务回滚机制边界

### 2.1 两层回滚架构

Secrets 的数据库操作处于**双层回滚保护**之下：

```
┌───────────────────────────────────────────────────────┐
│  L2: @safe_db_query 装饰器                             │
│  - 捕获 3 种可重试异常                                 │
│  - 执行 rollback() 重置事务状态                        │
│  - 最多重试 2 次后抛出                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │  L1: BaseModel.save/update/delete 方法内        │   │
│  │  - 捕获所有 Exception                            │   │
│  │  - 执行 rollback() 后立即抛出                    │   │
│  │  - 不重试，直接上抛                              │   │
│  └─────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

### 2.2 L1 回滚：BaseModel 内的事务保护

[BaseModel.save()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L67-L81)、[update()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L83-L93)、[delete()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L95-L102) 三个写操作均内置回滚：

```python
def save(self, commit=True) -> None:
    # ... 字段校验 ...
    self.session.add(self)
    if commit:
        try:
            self.session.commit()
        except Exception as e:
            self.session.rollback()  # ← 回滚点 L1
            raise e                  # ← 立即上抛，不重试
```

**L1 边界**：
- 捕获范围：**所有 Exception**（比 L2 宽得多）
- 回滚动作：`session.rollback()`
- 后续行为：**不重试**，直接把异常上抛给 L2

### 2.3 L2 回滚：safe_db_query 装饰器

[safe_db_query](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171) 的回滚逻辑：

```python
except (OperationalError, PendingRollbackError, InternalError) as e:
    db_connection.session.rollback()  # ← 回滚点 L2
    if retry_count >= DB_RETRY_COUNT:
        raise e
    retry_count += 1
```

**L2 边界**：
- 捕获范围：**仅 3 种异常**（比 L1 窄）
- 回滚动作：`session.rollback()`
- 后续行为：**重试 2 次**，仍失败则上抛

### 2.4 回滚调用链示例（创建 Secret 失败场景）

假设数据库连接临时断开（OperationalError）：

```
调用 create_secret(name, value)
  │
  ├─ _create_key_files(secrets_dir)  ← 文件操作，不在事务内
  │   └─ 创建 key 和 uuid 文件
  │
  ├─ Fernet 加密明文 → encrypted_value
  │
  └─ Secret(name=..., value=encrypted_value, ...).save()
       │
       ├─ session.add(secret)
       └─ try: session.commit()
          │
          └─ 失败 → OperationalError
              │
              ├─ L1 回滚: session.rollback()    ← 第一层回滚
              └─ raise e
                  │
                  └─ @safe_db_query 捕获
                      ├─ L2 回滚: session.rollback()  ← 第二层回滚（幂等）
                      ├─ retry_count = 1
                      └─ 重试：重新执行 create_secret() 整个函数
                          │
                          ├─ _create_key_files(secrets_dir)  ← 重新执行，已存在则读
                          └─ Secret(...).save()
                              └─ 成功 → 返回
```

### 2.5 关键边界：不在回滚范围内的操作

**文件系统操作不会回滚**：

```
create_secret() 执行时序：

  time →
    │
    ├─ ① _create_key_files()  ← 文件系统操作：创建目录、key文件、uuid文件
    │     ✓ 如果这一步失败：异常直接抛出，已创建的文件残留
    │
    ├─ ② Fernet 加密          ← 内存操作，无副作用
    │
    └─ ③ secret.save()        ← DB 操作：事务内
          ✓ 如果这一步失败：DB 回滚，但步骤 ① 创建的文件 保留！
```

**这是一个设计上的边界**：密钥文件一旦创建就不会因为后续 DB 操作失败而回滚删除。
- 好处：幂等友好，下次重试时直接复用已有密钥
- 风险：如果创建流程永久失败，会留下"孤儿"密钥文件（无对应 DB 记录）

---

## 三、加密解密容错边界

### 3.1 解密容错：InvalidToken 静默跳过

在 [get_secret_value](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L180-L183) 和 [get_valid_secrets_for_repo](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L148-L153) 中，解密失败的处理非常"宽容"：

```python
try:
    return fernet.decrypt(secret.value.encode('utf-8')).decode('utf-8')
except InvalidToken:
    pass  # ← 静默跳过，不抛出，不记录 error 日志
```

### 3.2 解密容错边界

**✅ 被静默吞掉的异常**：

| 异常 | 触发场景 | 处理方式 |
|------|---------|---------|
| `cryptography.fernet.InvalidToken` | 密钥不匹配、密文损坏、篡改 | `pass`，继续下一条 |

**❌ 会抛出的异常（未捕获）**：

| 异常 | 触发场景 | 行为 |
|------|---------|------|
| `TypeError` | `secret.value` 为 None 或非字符串 | 上抛异常，中断流程 |
| `ValueError` | Fernet key 格式非法 | 上抛异常 |
| 其他加密库异常 | 如 `cryptography.exceptions.UnsupportedAlgorithm` | 上抛异常 |

### 3.3 密钥文件读取容错

[_get_encryption_key](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L283-L310) 对两个密钥文件的读取异常完全屏蔽：

```python
try:
    with open(key_file, 'r', encoding='utf-8') as f:
        key = f.read()
except Exception:  # ← 捕获所有异常！
    key = None

try:
    with open(uuid_file, 'r', encoding='utf-8') as f:
        key_uuid = f.read()
except Exception:  # ← 捕获所有异常！
    key_uuid = None
```

**边界分析**：
- 捕获范围：**所有 Exception**（文件不存在、权限不足、磁盘错误等统统捕获）
- 处理方式：返回 `None`，不抛出
- 后续影响：`key is None` 时，`get_secret_value` 直接输出 WARNING 并返回 None

### 3.4 警告级别控制：suppress_warning 参数

`get_secret_value` 和 `delete_secret` 都支持 `suppress_warning` 参数：

```python
if not kwargs.get('suppress_warning', False):
    print(f'WARNING: Could not find secret value for secret {name}.')
```

**使用场景对比**：

| 使用方 | suppress_warning | 原因 |
|-------|-----------------|------|
| 普通业务调用 | False（默认） | 找不到 secret 是异常情况，需要提示 |
| [get_aws_value_from_secrets](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/services/aws/__init__.py#L11-L16) | True | 只是多源尝试中的一环，找不到是正常的 |
| [git utils](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/git/utils.py) | True | Git 密钥可能不存在，尝试多种获取方式 |

### 3.5 DB 安全边界：get_secret_value_db_safe

[get_secret_value_db_safe](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L204-L212) 是又一层容错：

```python
def get_secret_value_db_safe(name: str, **kwargs) -> Optional[str]:
    from mage_ai.orchestration.db import db_connection
    if db_connection.session and db_connection.session.is_active:
        return get_secret_value(name, **kwargs)
    else:
        return None  # DB 未初始化时直接返回 None
```

**边界**：在数据库连接还没建立好的阶段（如应用启动早期），调用这个版本不会出错。

---

## 四、配置后端降级边界

### 4.1 SettingsBackend 的三级降级

[SettingsBackend.get()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L26-L42) 实现了配置读取的三级回退：

```
调用 settings.get_value('DB_PASSWORD', default='fallback')
    │
    ├─ 第 1 级: _get(key)  ← 具体后端实现
    │   ├─ AWS 后端: 调用 AWS Secrets Manager API
    │   ├─ 默认后端: 直接返回 None
    │   └─ 返回 None → 继续降级
    │
    ├─ 第 2 级: os.getenv(key)  ← 环境变量
    │   └─ 不存在 → 继续降级
    │
    └─ 第 3 级: kwargs.get('default')  ← 调用方指定的默认值
        └─ 未提供 default → 返回 None
```

**每一级的边界**：

| 级别 | 来源 | 失败时行为 |
|------|------|-----------|
| L1 后端 | AWS Secrets Manager / 其他 | 返回 None，不抛出（ResourceNotFoundException 被吞） |
| L2 环境变量 | 操作系统环境变量 | 返回 None，天然静默 |
| L3 默认值 | 调用方传入的 `default` 参数 | 返回 None |

### 4.2 AWSSecretsManagerBackend 的错误边界

[AWSSecretsManagerBackend._get()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L77-L96) 对 AWS API 错误的处理：

```python
try:
    if self.cache is not None:
        return self.cache.get_secret_string(key)
    else:
        secret_response = self.client.get_secret_value(SecretId=key)
        # ... 处理 SecretBinary / SecretString ...
except ClientError as error:
    if error.response['Error']['Code'] != 'ResourceNotFoundException':
        logger.exception('Failed to get secret %s from AWS Secrets Manager.', key)
    return None
```

**错误分类处理**：

| 错误类型 | 处理方式 | 边界行为 |
|---------|---------|---------|
| `ResourceNotFoundException` | 静默返回 None | secret 不存在视为正常，不告警 |
| 其他 ClientError（权限、网络等） | 记录 exception 日志 + 返回 None | 降级处理，不中断流程 |
| 非 ClientError 异常（如 import 失败） | 向上抛出 | 严重错误，暴露给上层 |

### 4.3 IO Config Loader 的降级边界

#### AWSSecretLoader 的错误处理

[AWSSecretLoader.__get_secret()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/io/config.py#L271-L311)：

```python
try:
    return self.client.get_secret_value(**secret_kwargs)
except ClientError as error:
    if error.response['Error']['Code'] == 'ResourceNotFoundException':
        return None  # ← 不存在：静默返回 None
    raise RuntimeError(  # ← 其他错误：抛出 RuntimeError
        f'Error loading config: {error.response["Error"]["Message"]}')
```

> **与 SettingsBackend 的区别**：这里对非 ResourceNotFound 的错误**直接抛出**，而不是降级返回 None。这是 IO Config 和 Settings Backend 的设计差异。

### 4.4 AWS 服务凭据的多源降级

[get_aws_access_key_id](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/services/aws/__init__.py#L32-L36) 等函数实现了 AWS 凭据的多源查找：

```python
def get_aws_access_key_id() -> Optional[str]:
    # 第 1 级：环境变量
    aws_access_key_id = os.getenv(AWS_ACCESS_KEY_ID)
    
    # 第 2 级：Mage Secrets（DB 中存储的）
    if aws_access_key_id is None:
        aws_access_key_id = get_aws_value_from_secrets(AWS_ACCESS_KEY_ID)
    
    # 都没有就返回 None
    return aws_access_key_id
```

`get_aws_value_from_secrets` 内部还有一层容错：

```python
def get_aws_value_from_secrets(name: str) -> Optional[str]:
    try:
        from mage_ai.data_preparation.shared.secrets import get_secret_value_db_safe
        return get_secret_value_db_safe(name, suppress_warning=True)
    except Exception:  # ← 捕获所有异常
        return None
```

**完整降级链**：

```
获取 AWS_ACCESS_KEY_ID
    │
    ├─ ① 环境变量 os.getenv('AWS_ACCESS_KEY_ID')
    │   └─ 不存在 → 继续
    │
    ├─ ② Mage Secrets DB 查找 (get_secret_value_db_safe)
    │   ├─ DB 未初始化 → 返回 None
    │   ├─ DB 查询异常（safe_db_query 重试耗尽）→ 被外层 try-catch 捕获
    │   └─ 找不到 secret → 返回 None
    │
    └─ ③ 都失败 → 返回 None（boto3 会继续尝试其他默认凭据链）
```

### 4.5 模板变量中的降级

[get_template_vars_no_db](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/utils.py#L16-L29) 中，各云厂商的 secret 变量是按需加载的：

```python
kwargs = dict(env_var=os.getenv, json_value=get_json_value)

try:
    from mage_ai.services.aws.secrets_manager.secrets_manager import get_secret
    kwargs['aws_secret_var'] = get_secret
except Exception:  # ← 导入失败就不提供该功能
    pass

try:
    from mage_ai.services.azure.key_vault.key_vault import get_secret
    kwargs['azure_secret_var'] = get_secret
except Exception:
    pass
```

**边界**：如果某云厂商的 SDK 没装，对应的模板函数就不注入，用户调用时会得到 `NameError` 而不是系统崩溃。

### 4.6 Azure Key Vault 的边界

[Azure Key Vault 实现](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/services/azure/key_vault/key_vault.py) 相对简单，容错较少：

```python
def get_secret(secret_name: str) -> str:
    return secrets_manager.client.get_secret(secret_name).value
```

**边界分析**：
- 初始化时 `AZURE_KEY_VAULT_URL` 缺失会抛 `Exception`
- 调用时 secret 不存在或权限不足会直接抛出 Azure SDK 的异常
- **没有**像 AWS 那样的 ResourceNotFoundException 静默处理

---

## 五、回退边界总览图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Secrets 调用入口                               │
│  (API / 模板变量 / IO Config / 业务代码)                          │
└─────────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        ▼                                     ▼
┌───────────────────────┐             ┌───────────────────────┐
│   配置后端降级         │             │   业务逻辑层           │
│   (SettingsBackend)   │             │   (shared/secrets.py)  │
│  - 三级回退           │             │  - 加密解密容错        │
│  - 静默降级           │             │  - InvalidToken 跳过   │
└───────────────────────┘             │  - 密钥文件容错        │
        ▲                             │  - suppress_warning    │
        │                             └───────────┬───────────┘
        │                                         │
        │                                         ▼
        │                             ┌───────────────────────┐
        │                             │   数据库访问层         │
        │                             │  - @safe_db_query     │
        │                             │  - 3 种异常重试 2 次   │
        │                             │  - 事务 rollback       │
        │                             └───────────┬───────────┘
        │                                         │
        │                                         ▼
        │                             ┌───────────────────────┐
        │                             │   BaseModel 内部      │
        │                             │  - save/update/delete │
        │                             │  - 所有异常 rollback   │
        │                             │  - 不重试，直接上抛     │
        │                             └───────────────────────┘
        │                                         │
        └─────────────────────────────────────────┘
                     各层独立，互不替代
```

---

## 六、关键代码索引

| 回退类型 | 核心代码位置 | 行号 |
|---------|------------|------|
| 数据库重试装饰器 | [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171) | L155-L171 |
| BaseModel 事务回滚 | [base.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L67-L102) | L67-L102 |
| 解密 InvalidToken 容错 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L148-L153) | L148-L153 |
| 密钥文件读取容错 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L296-L306) | L296-L306 |
| DB 安全调用包装 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L204-L212) | L204-L212 |
| SettingsBackend 三级降级 | [backends.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L26-L42) | L26-L42 |
| AWS 后端错误处理 | [backends.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/settings/backends.py#L77-L96) | L77-L96 |
| AWS IO Loader 错误处理 | [io/config.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/io/config.py#L305-L311) | L305-L311 |
| AWS 凭据多源降级 | [aws/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/services/aws/__init__.py#L11-L50) | L11-L50 |
| 模板变量按需加载 | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/utils.py#L16-L29) | L16-L29 |

---

## 七、边界要点总结

### 数据库重试
- **仅重试 3 种异常**：OperationalError / PendingRollbackError / InternalError
- **重试 2 次**，每次重试前先 rollback
- **重试范围**：被装饰函数的**全部代码**会重新执行（包括文件操作等非 DB 操作）

### 事务回滚
- **两层回滚**：BaseModel 内（捕获所有异常） + safe_db_query（捕获 3 种 + 重试）
- **回滚范围**：仅数据库会话内的操作
- **回滚不了的**：文件系统操作（密钥文件创建）、内存变量、外部 API 调用

### 加密解密容错
- **只吞 InvalidToken**：其他加密异常照常抛出
- **密钥文件读取**：所有异常都吞，返回 None
- **找不到 secret**：默认打 WARNING，可用 `suppress_warning=True` 静默
- **DB 未就绪**：`get_secret_value_db_safe` 直接返回 None，不报错

### 配置后端降级
- **SettingsBackend 三级**：后端 API → 环境变量 → default 参数，全部静默降级
- **AWS 后端**：ResourceNotFound 静默，其他错误打日志后降级
- **IO Config Loader**：ResourceNotFound 静默，**其他错误抛出**（与 SettingsBackend 不同）
- **AWS 凭据查找**：环境变量 → Mage Secrets DB → 无（交给 boto3 默认链）
- **模板变量**：云 SDK 导入失败不提供该函数，功能降级而非崩溃

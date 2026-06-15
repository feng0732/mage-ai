# Secrets 管理：装饰器重试范围 vs 事务回滚范围

## 核心结论

**装饰器重试范围 ≠ 事务回滚范围**，这是两个完全不同维度的概念：

| 维度 | 装饰器重试（@safe_db_query） | 事务回滚（session.rollback） |
|------|---------------------------|---------------------------|
| **作用范围** | 整个被装饰函数的所有代码 | 仅数据库会话内的操作 |
| **触发条件** | 3 种 SQLAlchemy 异常 | commit 时任何异常 |
| **行为** | rollback + 重新执行整个函数 | 撤销未提交的 DB 变更 |
| **覆盖的操作** | 文件操作 + 加密 + DB 查询 + DB 写入 | 只有 DB 变更 |
| **失败后状态** | 文件已创建、DB 已回滚 | DB 回到事务前状态 |

---

## 一、两个范围的边界图示

以 `create_secret` 为例，我们用方框标出两个范围的边界：

```
┌──────────────────────────────────────────────────────────────────┐
│  装饰器重试范围（@safe_db_query）                                 │
│  整个 create_secret 函数体都会被重新执行                           │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌─────────────────────────────┐  │
│  │ 步骤1    │   │ 步骤2    │   │ 步骤3                       │  │
│  │ 参数校验 │   │ 密钥文件  │   │ Fernet 加密 → DB 写入       │  │
│  │ 抛异常   │   │ 创建      │   │                             │  │
│  └──────────┘   └──────────┘   │  ┌───────────────────────┐  │  │
│       │              │         │  │ 事务回滚范围          │  │  │
│       ▼              ▼         │  │ (session.rollback)    │  │  │
│   直接抛出       直接抛出       │  └───────────────────────┘  │  │
│   不重试         不重试         │       ▲                      │  │
│                                │       │                      │  │
│                                └───────┼──────────────────────┘  │
│                                        │                         │
│                                        ▼                         │
│                              捕获 3 种 DB 异常                    │
│                              rollback 后重试                       │
└──────────────────────────────────────────────────────────────────┘
```

关键结论：
- **装饰器**包了整个函数，但**只有 DB 相关的 3 种异常**才会触发重试
- **事务回滚**只发生在数据库会话内，文件系统操作**不在回滚范围内**

---

## 二、逐行拆解 create_secret 的执行路径

我们以 [create_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L87-L126) 为例，逐行标注两个范围的边界。

### 代码标注版

```python
@safe_db_query                    # ← 装饰器起点：重试范围的最外层
def create_secret(
    name: str,
    value: str,
    entity: Entity = Entity.GLOBAL,
    project_uuid: str = None,
    pipeline_uuid: str = None,
    repo_name: str = None,
):
    # ============ 区域 A: 参数校验 ============
    # ↓ 不在事务内 ↓ 不在重试触发范围（抛普通 Exception）
    from mage_ai.orchestration.db.models.secrets import Secret
    missing_values = []
    if entity in [Entity.PROJECT, Entity.PIPELINE] and not project_uuid:
        missing_values.append('project_uuid')
    if entity == Entity.PIPELINE and not pipeline_uuid:
        missing_values.append('pipeline_uuid')
    if missing_values:
        raise Exception(...)     # ← 抛普通异常 → 装饰器不捕获 → 不重试

    # ============ 区域 B: 密钥目录计算 ============
    # ↓ 纯函数计算 ↓ 无副作用 ↓ 不在事务内
    secrets_dir = get_secrets_dir(entity, project_uuid=..., pipeline_uuid=...)

    # ============ 区域 C: 密钥文件创建 ============
    # ↓ 文件系统操作 ↓ 不在事务内 ↓ 不在重试触发范围
    key, key_uuid = _create_key_files(secrets_dir)
    # ↑ 如果这里失败（如权限不足、磁盘满），异常直接抛出，装饰器不捕获，不重试

    # ============ 区域 D: Fernet 加密 ============
    # ↓ 纯内存操作 ↓ 无副作用 ↓ 不在事务内
    fernet = Fernet(key)
    encrypted_value = fernet.encrypt(value.encode('utf-8')).decode('utf-8')
    kwargs = {
        'name': name,
        'value': encrypted_value,
        'key_uuid': key_uuid,
        'repo_name': repo_name or Project().repo_path_for_database_query('secrets')[0],
    }

    # ============ 区域 E: 数据库写入 ============
    # ↓ 进入事务范围 ↓
    secret = Secret(**kwargs)
    secret.save()              # ← 这里面有自己的 try-catch rollback
    # ↑ save() 内部：session.add → try: session.commit
    #                       ↓
    #                   失败 → session.rollback() → raise e
    #                       ↓
    #                被 @safe_db_query 捕获
    #                如果是 3 种异常之一 → rollback + 重试
    #                其他异常 → 直接抛出

    return secret
# ← 装饰器终点
```

### 各区域失败行为对照表

| 区域 | 操作类型 | 失败时异常类型 | 装饰器是否捕获 | 是否重试 | 回滚什么 |
|------|---------|-------------|--------------|---------|---------|
| A 参数校验 | 逻辑判断 | `Exception` | ❌ 否 | ❌ 否 | 无 |
| B 目录计算 | 纯计算 | （几乎不会失败） | - | - | 无 |
| C 密钥文件创建 | 文件系统 I/O | `OSError` 等 | ❌ 否 | ❌ 否 | **已创建的文件残留** |
| D Fernet 加密 | 内存计算 | `ValueError`/`TypeError` | ❌ 否 | ❌ 否 | 无 |
| E DB 写入 | 数据库事务 | `OperationalError` 等 3 种 | ✅ 是 | ✅ 是 | **DB 回滚** |
| E DB 写入 | 数据库事务 | 其他异常（如 IntegrityError） | ❌ 否 | ❌ 否 | **DB 回滚**（L1 层） |

---

## 三、密钥文件创建怎么走？

### 3.1 _create_key_files 的执行路径

[_create_key_files](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L247-L280) 的内部流程：

```
进入 _create_key_files(secrets_dir)
    │
    ├─ 步骤 1: 检查目录是否存在
    │   ├─ 不存在 → os.makedirs(secrets_dir) 创建
    │   │   └─ 失败（权限不足/磁盘满）→ OSError 直接抛出
    │   └─ 已存在 → 跳过
    │
    ├─ 步骤 2: 处理 key 文件
    │   ├─ key 文件已存在 → open 读取 → key 变量
    │   │   └─ 读取失败 → 异常抛出
    │   └─ key 文件不存在 → Fernet.generate_key() 生成 → open 写入
    │       └─ 写入失败 → 异常抛出，已生成的目录保留
    │
    └─ 步骤 3: 处理 uuid 文件
        ├─ uuid 文件已存在 → open 读取 → key_uuid 变量
        └─ uuid 文件不存在 → uuid.uuid4() 生成 → open 写入
            └─ 写入失败 → 异常抛出，已创建的目录和 key 文件保留
```

### 3.2 部分失败的状态

`_create_key_files` **没有原子性保证**，中间任何一步失败，已经完成的操作**不会回滚**：

| 失败发生在 | 已创建的内容 | 后续重试时的行为 |
|-----------|-------------|----------------|
| `os.makedirs` 失败 | 什么都没创建 | 再次尝试创建目录 |
| key 文件写入失败 | 目录已创建 | 目录已存在跳过，重新尝试生成和写入 key |
| uuid 文件写入失败 | 目录 + key 文件已创建 | 目录和 key 都存在跳过，重新尝试生成和写入 uuid |

### 3.3 关键边界：文件操作不在重试保护内

虽然 `_create_key_files` 处于 `@safe_db_query` 装饰器的包裹范围内（语法上在函数体内），但**文件系统异常不会触发重试**，因为装饰器只捕获 3 种 SQLAlchemy 异常。

```python
@safe_db_query
def create_secret(...):
    ...
    key, key_uuid = _create_key_files(secrets_dir)  # ← 语法上在装饰器内
    ...                                               #     但文件异常不会被捕获
    secret.save()                                     # ← 只有这里的 DB 异常可能触发重试
```

**类比**：就像下雨时你站在伞下，但只有头部被遮住——你的身体（DB 操作）在伞下，但脚（文件操作）露在外面。

---

## 四、数据库写入失败怎么走？

### 4.1 两层回滚架构

数据库写入失败时，会经过**两层回滚**，每层的职责不同：

```
secret.save()
  │
  └─ session.add(secret)           ← 加入会话缓存，还没写入 DB
       │
       └─ try: session.commit()    ← 真正写入数据库
           │
           ├─ ✅ 成功 → 正常返回
           │
           └─ ❌ 失败（抛出异常 e）
                │
                ├─ L1 回滚（BaseModel 内）
                │   ├─ session.rollback()  ← 第一层回滚
                │   └─ raise e              ← 原样上抛
                │
                └─ L2 回滚（@safe_db_query 内）
                     │
                     ├─ 是 3 种可重试异常吗？
                     │   ├─ 是 → 
                     │   │   ├─ session.rollback()  ← 第二层回滚（幂等）
                     │   │   ├─ retry_count++
                     │   │   └─ 重试 < 2 次 → 重新执行整个 create_secret
                     │   │
                     │   └─ 否 →
                     │       └─ （不捕获，异常继续向上抛）
                     │
                     └─ 异常到了调用方
```

### 4.2 L1 vs L2 回滚的区别

| 特性 | L1 回滚（BaseModel.save 内） | L2 回滚（@safe_db_query） |
|------|---------------------------|--------------------------|
| 触发位置 | [BaseModel.save()](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L67-L81) L77-L81 | [safe_db_query](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171) L161-L169 |
| 捕获范围 | **所有 Exception** | 仅 3 种 SQLAlchemy 异常 |
| 回滚动作 | `session.rollback()` | `session.rollback()` |
| 回滚后 | 立即上抛异常 | 可能重试 |
| 重试？ | ❌ 不重试 | ✅ 重试最多 2 次 |
| 覆盖的写操作 | save / update / delete 各管各的 | 整个被装饰函数内的所有 DB 操作 |

### 4.3 两次 rollback 会不会有问题？

**不会**。`session.rollback()` 是幂等的——对已经回滚的会话再次调用 rollback 不会报错。

```
第一次失败:
  session.commit() 失败
  → L1: session.rollback()    事务回到初始状态
  → raise OperationalError
  → L2: 捕获异常
     → session.rollback()     已经是干净状态，无副作用
     → 重试...
```

### 4.4 什么异常只触发 L1 不触发 L2？

**除了 3 种可重试异常之外的所有 DB 异常**，都会只经过 L1 回滚然后直接抛出：

| 异常 | L1 回滚 | L2 捕获重试 | 举例场景 |
|------|---------|-----------|---------|
| `IntegrityError` | ✅ | ❌ | 同名 secret 已存在（唯一约束冲突） |
| `DataError` | ✅ | ❌ | value 太长超出 Text 列限制 |
| `ProgrammingError` | ✅ | ❌ | 表不存在（代码/迁移问题） |
| `OperationalError` | ✅ | ✅ | 数据库连接断开 |
| `PendingRollbackError` | ✅ | ✅ | 前一个事务失败了 |
| `InternalError` | ✅ | ✅ | 死锁 |

---

## 五、重试时重新执行怎么走？

### 5.1 重试的完整路径

当 `OperationalError` 触发重试时，**整个 `create_secret` 函数会从头重新执行**，而不是从失败的那一行继续。

让我们追踪第一次执行和第一次重试的完整路径：

```
第 1 次执行 create_secret('DB_PASS', 'secret123')
  │
  ├─ 参数校验 → 通过
  ├─ get_secrets_dir() → 计算出路径
  ├─ _create_key_files(secrets_dir)
  │   ├─ 目录不存在 → os.makedirs 创建 ✅
  │   ├─ key 文件不存在 → 生成并写入 ✅
  │   └─ uuid 文件不存在 → 生成并写入 ✅
  ├─ Fernet 加密 → 得到 encrypted_value
  ├─ 构造 Secret 对象
  └─ secret.save()
       ├─ session.add(secret)
       └─ session.commit()
          └─ 失败！连接断开 → OperationalError
             ├─ L1: session.rollback()
             └─ raise e → 被 @safe_db_query 捕获
                ├─ session.rollback()
                ├─ retry_count = 1
                └─ ↓ 重 ↓ 试 ↓

第 2 次执行 create_secret('DB_PASS', 'secret123')  ← 整个函数从头再来！
  │
  ├─ 参数校验 → 通过（再来一遍）
  ├─ get_secrets_dir() → 同样的路径（再来一遍）
  ├─ _create_key_files(secrets_dir)
  │   ├─ 目录已存在 → 跳过（因为第 1 次已经创建了）
  │   ├─ key 文件已存在 → 读取（不是重新生成！）
  │   └─ uuid 文件已存在 → 读取（不是重新生成！）
  ├─ Fernet 加密 → 同样的明文 + 同样的 key = 同样的密文？
  │                   ↑ 注意：Fernet 每次加密结果不同，但用同一个 key 可以解密
  ├─ 构造 Secret 对象
  └─ secret.save()
       ├─ session.add(secret)
       └─ session.commit()
          └─ 成功 → 返回 secret 对象
```

### 5.2 重试时的状态残留

重试不是时光倒流，**已经发生的副作用不会消失**。以下是重试时各阶段的状态：

| 阶段 | 第 1 次执行后 | 第 2 次（重试）执行时 | 行为 |
|------|-------------|---------------------|------|
| 目录 | 已创建 | 已存在 | `os.makedirs` 跳过（因为先判断 `if not os.path.exists`） |
| key 文件 | 已创建 + 已写入 | 已存在 | 读取已有内容，不重新生成 |
| uuid 文件 | 已创建 + 已写入 | 已存在 | 读取已有内容，不重新生成 |
| DB 中的 secret 记录 | 不存在（已回滚） | 不存在 | 重新 add + commit |
| Fernet 加密 | 内存值，已消失 | 重新计算 | 用相同 key 和明文，但密文会不同（Fernet 有随机 IV） |

### 5.3 关键性质：幂等性

虽然密文不同，但重试不会造成重复数据或冲突，因为：

1. **密钥文件是幂等的**：已存在就直接读取，不会重复创建
2. **secret 名称是唯一的**：如果第一次的 DB 记录已经回滚，第二次写入同名 secret 不会冲突
3. **key_uuid 不变**：因为用的是同一个密钥文件，所以 key_uuid 相同，唯一约束不会冲突

> **但有一个例外**：如果第一次执行在 `secret.save()` 之前失败（比如加密时出错），而且文件已经创建好了，那么文件会变成"孤儿"——没有对应的 DB 记录，但密钥文件还在。不过这不影响后续使用，因为下次创建任意 secret 时都会复用这个密钥。

---

## 六、delete_secret 的重试与回滚分析

作为对比，我们快速看一下 [delete_secret](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L215-L244) 的路径：

```python
@safe_db_query
def delete_secret(name, ...):
    # 区域 1: 读取密钥文件（文件 I/O，不在事务内）
    _, key_uuid = _get_encryption_key(...)
    # ↑ 异常被吞掉，返回 None（_get_encryption_key 内部 try-catch）

    # 区域 2: 查询新格式记录（DB 读，在会话内）
    if key_uuid:
        secret = Secret.get_secret(name, key_uuid)

    # 区域 3: 回退查询旧格式记录（DB 读，在会话内）
    if entity == GLOBAL and not secret:
        secret = Secret.query.filter(... key_uuid IS NULL).one_or_none()

    # 区域 4: 删除（DB 写，在事务内）
    if secret:
        secret.delete()   # ← 内部有 try-catch rollback（L1）
        # ↑ 如果这里触发 OperationalError，会被装饰器重试（L2）
```

**delete_secret 的特点**：
- 前面的文件读取被 `_get_encryption_key` 内部吞了异常，不会触发重试
- DB 查询失败（3 种异常）会触发重试，整个函数从头执行
- DB 删除失败（3 种异常）会触发重试，整个函数从头执行
- 同样：文件操作不在回滚范围，DB 操作在回滚范围

---

## 七、关键代码索引

| 概念 | 文件 | 行号 |
|------|------|------|
| @safe_db_query 装饰器 | [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L171) | L155-L171 |
| BaseModel.save 内的回滚 | [base.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/models/base.py#L67-L81) | L67-L81 |
| create_secret 函数 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L87-L126) | L87-L126 |
| _create_key_files 函数 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L247-L280) | L247-L280 |
| _get_encryption_key 函数 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L283-L311) | L283-L311 |
| delete_secret 函数 | [secrets.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/data_preparation/shared/secrets.py#L215-L244) | L215-L244 |
| DB_RETRY_COUNT 常量 | [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/316-mage-ai/mage_ai/orchestration/db/__init__.py#L19) | L19 |

---

## 八、总结：核心区别速记

### 装饰器重试范围
- **是什么**：`@safe_db_query` 装饰器包裹的整个函数体
- **做什么**：捕获 3 种 DB 异常，回滚事务，重新执行整个函数
- **管什么**：数据库连接性、死锁等**临时性**故障
- **不管什么**：文件操作、逻辑错误、数据错误
- **副作用残留**：文件系统操作的结果在重试之间保持

### 事务回滚范围
- **是什么**：数据库会话（session）内的所有未提交变更
- **做什么**：`session.rollback()` 撤销未提交的 insert/update/delete
- **管什么**：数据库中的数据变更
- **不管什么**：文件系统、内存变量、外部 API 调用
- **副作用残留**：回滚后 DB 回到事务开始前的状态，完全干净

### 一句话总结

> **装饰器重试是"整个函数再来一遍"，事务回滚是"数据库撤回上一步"。两者叠加时，文件操作的副作用在重试之间保留，数据库的副作用每次都被清空重来。**

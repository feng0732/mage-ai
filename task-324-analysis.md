# Mage AI 全局 Variable 与运行参数 —— 代码全景图

## 一、模块分工总览

Mage AI 的全局变量与运行参数分散在多个模块中，按职责可划分为 **6 层**：

```
┌─────────────────────────────────────────────────────────┐
│  CLI 入口层        mage_ai/cli/main.py                  │
├─────────────────────────────────────────────────────────┤
│  服务器启动层      mage_ai/server/server.py             │
│                   mage_ai/server/setup.py               │
│                   mage_ai/server/scheduler_manager.py   │
├─────────────────────────────────────────────────────────┤
│  设置中心层        mage_ai/settings/                    │
│                   ├── server.py   ← 全局运行参数的单一真相源   │
│                   ├── repo.py     ← 仓库路径与变量目录      │
│                   ├── backends.py ← 设置后端抽象            │
│                   ├── constants.py← 元数据文件名/环境变量名  │
│                   ├── keys/auth.py← 认证/ Git 环境变量名    │
│                   └── __init__.py ← Settings 单例         │
├─────────────────────────────────────────────────────────┤
│  仓库配置层        mage_ai/data_preparation/repo_manager.py  │
│                   (RepoConfig 从 metadata.yaml 加载)          │
├─────────────────────────────────────────────────────────┤
│  编排/DB 层        mage_ai/orchestration/db/__init__.py      │
│                   mage_ai/orchestration/constants.py          │
├─────────────────────────────────────────────────────────┤
│  基础设施层        mage_ai/shared/constants.py  (InstanceType)│
│                   mage_ai/shared/environments.py (ENV 判断)   │
│                   mage_ai/cluster_manager/constants.py (ClusterType)│
│                   mage_ai/cache/constants.py (缓存键/目录)    │
└─────────────────────────────────────────────────────────┘
```

---

## 二、全局运行参数详解（settings/server.py）

[server.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/server.py) 是 Mage 运行参数的 **单一真相源**，所有参数均通过 `os.getenv()` 在模块加载时读取，并以模块级变量暴露。按功能域分组如下：

### 2.1 调试与环境

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `DEBUG` | `DEBUG` | `False` | 调试模式开关 |
| `DEBUG_MEMORY` | `DEBUG_MEMORY` | `0` | 内存调试 |
| `DEBUG_FILE_IO` | `DEBUG_FILE_IO` | `0` | 文件 IO 调试 |
| `HIDE_ENV_VAR_VALUES` | `HIDE_ENV_VAR_VALUES` | `1` | 隐藏环境变量值（安全） |

### 2.2 认证与令牌

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `REQUIRE_USER_AUTHENTICATION` | `REQUIRE_USER_AUTHENTICATION` | `True` | 是否强制用户认证 |
| `REQUIRE_USER_PERMISSIONS` | `REQUIRE_USER_PERMISSIONS` | `False` | 是否启用权限控制 |
| `AUTHENTICATION_MODE` | `AUTHENTICATION_MODE` | `'LOCAL'` | 认证模式（LOCAL/LDAP/OAUTH） |
| `JWT_SECRET` | `JWT_SECRET` | `'materia'` | JWT 签名密钥 |
| `OAUTH2_APPLICATION_CLIENT_ID` | 硬编码 | `'zkWlN0Pk...'` | 前端 OAuth 客户端 ID |
| `MAGE_ACCESS_TOKEN_EXPIRY_TIME` | `MAGE_ACCESS_TOKEN_EXPIRY_TIME` | `2592000` | 访问令牌过期时间（秒） |

### 2.3 服务器行为

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `DISABLE_AUTO_BROWSER_OPEN` | `DISABLE_AUTO_BROWSER_OPEN` | `False` | 启动时不自动打开浏览器 |
| `DISABLE_AUTORELOAD` | `DISABLE_AUTORELOAD` | `False` | 禁用 Tornado 自动重载 |
| `SERVER_VERBOSITY` | `SERVER_VERBOSITY` | `'info'` | 日志级别 |
| `SERVER_LOGGING_FORMAT` | `SERVER_LOGGING_FORMAT` | `'plaintext'` | 日志格式 |
| `REDIS_URL` | `REDIS_URL` | `None` | Redis 连接 URL |
| `CONCURRENCY_CONFIG_BLOCK_RUN_LIMIT` | `CONCURRENCY_CONFIG_BLOCK_RUN_LIMIT` | `None` | Block 并发上限 |
| `CONCURRENCY_CONFIG_PIPELINE_RUN_LIMIT` | `CONCURRENCY_CONFIG_PIPELINE_RUN_LIMIT` | `None` | Pipeline 并发上限 |

### 2.4 路径与 URL

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `BASE_PATH` | `MAGE_BASE_PATH` | `None` | 前后端统一基础路径 |
| `REQUESTS_BASE_PATH` | `MAGE_REQUESTS_BASE_PATH` | `BASE_PATH` | 前端请求基础路径 |
| `ROUTES_BASE_PATH` | `MAGE_ROUTES_BASE_PATH` | `BASE_PATH` | 后端路由基础路径 |
| `MAGE_PUBLIC_HOST` | `MAGE_PUBLIC_HOST` | `http://localhost:6789` | 公网访问地址 |
| `LOGS_DIR_PATH` | `LOGS_DIR_PATH` | `None` | 日志目录 |

### 2.5 数据处理与内存

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `DYNAMIC_BLOCKS_VERSION` | `DYNAMIC_BLOCKS_VERSION` | `1` | 动态 Block 版本 |
| `MEMORY_MANAGER_VERSION` | `MEMORY_MANAGER_VERSION` | `1` | 内存管理器版本 |
| `MEMORY_MANAGER_PANDAS_VERSION` | `MEMORY_MANAGER_PANDAS_VERSION` | `1` | Pandas 内存管理版本 |
| `MEMORY_MANAGER_POLARS_VERSION` | `MEMORY_MANAGER_POLARS_VERSION` | `1` | Polars 内存管理版本 |
| `VARIABLE_DATA_OUTPUT_META_CACHE` | `VARIABLE_DATA_OUTPUT_META_CACHE` | `0` | 变量输出元数据缓存 |

### 2.6 调度器

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `SCHEDULER_TRIGGER_INTERVAL` | `SCHEDULER_TRIGGER_INTERVAL` | `10` | 调度器轮询间隔（秒） |

### 2.7 可观测性

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `SENTRY_DSN` | `SENTRY_DSN` | `None` | Sentry DSN |
| `ENABLE_PROMETHEUS` | `ENABLE_PROMETHEUS` | `False` | Prometheus 指标 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `OTEL_EXPORTER_OTLP_ENDPOINT` | `None` | OpenTelemetry 端点 |

### 2.8 IDE 与 Kernel

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `KERNEL_MANAGER` | `KERNEL_MANAGER` | `'default'` | Kernel 管理器类型 |
| `KERNEL_MAGIC` | 计算值 | `False` | 是否使用 magic kernel |

### 2.9 Pipeline 初始化

| 变量 | 环境变量 | 默认值 | 说明 |
|------|---------|--------|------|
| `RUN_PIPELINE_IN_ONE_PROCESS` | `RUN_PIPELINE_IN_ONE_PROCESS` | `False` | 单进程运行 Pipeline |
| `RESTART_STREAMING_PIPELINES_ON_REQUIREMENTS_CHANGE` | `MAGE_RESTART_STREAMING_PIPELINES_ON_REQUIREMENTS_CHANGE` | `False` | 依赖变更时重启流式 Pipeline |

---

## 三、Settings 单例与后端抽象

### 3.1 Settings 类（settings/__init__.py）

[__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/__init__.py) 定义了全局单例 `settings = Settings()`：

```python
class Settings():
    def __init__(self):
        self.settings_backend = SettingsBackend()

    def set_settings_backend(self, backend_type=None, **kwargs):
        if backend_type == BackendType.AWS_SECRETS_MANAGER:
            self.settings_backend = AWSSecretsManagerBackend(**kwargs)
        else:
            self.settings_backend = SettingsBackend(**kwargs)

    def get_value(self, key, default=None):
        return self.settings_backend.get(key, default=default)
```

### 3.2 后端查找优先级链

[backends.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/backends.py) 中 `SettingsBackend.get()` 的查找顺序：

```
1. 后端私有存储（AWS Secrets Manager / 本地 None）
2. os.getenv(key)              ← 环境变量
3. kwargs.get('default')       ← 调用方默认值
```

### 3.3 MAGE_SETTINGS_ENVIRONMENT_VARIABLES 列表

[server.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/server.py#L253-L314) 底部维护了 `MAGE_SETTINGS_ENVIRONMENT_VARIABLES` 列表（约 50+ 项），这些环境变量会在工作区间复制时同步迁移，是 Mage 集群多工作区管理的核心数据清单。

---

## 四、仓库路径与变量目录

### 4.1 仓库路径解析链

[repo.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/repo.py) 和 [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/utils.py) 共同构成仓库路径体系：

```
MAGE_REPO_PATH 环境变量  →  base_repo_path()  →  os.getcwd()
         ↓
set_repo_path(project)   →  os.environ['MAGE_REPO_PATH'] = project
                            sys.path.append(dirname(project))
                            set_project_platform_activated_flag()
```

`get_repo_path()` 的完整判断流程：

```
1. root_project=True → 直接返回 base_repo_path()
2. root_project=False:
   a. project_platform_activated()?
      i.  提供了 file_path? → get_repo_paths_for_file_path() 定位
      ii. 否则 → build_active_project_repo_path() 构建活跃项目路径
   b. 未激活多项目平台 → 直接返回 base_repo_path()
3. absolute_path=False → 计算 Path.relative_to()
```

### 4.2 变量目录解析链

[get_variables_dir()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/settings/repo.py#L162-L225) 的优先级：

```
1. os.getenv('MAGE_DATA_DIR')           ← 环境变量优先
2. repo_config['variables_dir']         ← metadata.yaml 配置
3. metadata.yaml 文件中的 variables_dir  ← 从文件读取（支持 Jinja2 模板渲染）
4. DEFAULT_MAGE_DATA_DIR (~/.mage_data) ← 兜底默认
```

变量目录还支持 `s3://` 和 `gs://` 远程存储前缀。

---

## 五、运行实例类型与项目类型

### 5.1 InstanceType（shared/constants.py）

[constants.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/shared/constants.py) 定义了实例部署模式：

| 类型 | 值 | 说明 |
|------|---|------|
| `SERVER_AND_SCHEDULER` | `'server_and_scheduler'` | Web + 调度器一体 |
| `SCHEDULER` | `'scheduler'` | 仅调度器（status_only 模式启动 Web） |
| `WEB_SERVER` | `'web_server'` | 仅 Web 服务器 |

由环境变量 `INSTANCE_TYPE` 控制，在 [start_server()](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/server/server.py#L756-L834) 中决定启动行为。

### 5.2 ProjectType（data_preparation/repo_manager.py）

[repo_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/repo_manager.py#L39-L42) 定义：

| 类型 | 值 | 说明 |
|------|---|------|
| `MAIN` | `'main'` | 主项目（管理多工作区） |
| `SUB` | `'sub'` | 子项目 |
| `STANDALONE` | `'standalone'` | 独立项目（默认） |

### 5.3 ClusterType（cluster_manager/constants.py）

[constants.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/cluster_manager/constants.py) 定义：

| 类型 | 值 |
|------|---|
| `EMR` | `'emr'` |
| `ECS` | `'ecs'` |
| `CLOUD_RUN` | `'cloud_run'` |
| `K8S` | `'k8s'` |

---

## 六、启动时序全景图

以下是 `mage start` 命令触发的完整启动时序：

```
┌───────────────────────────────────────────────────────────────┐
│ Phase 1: CLI 入口 (cli/main.py start)                         │
│                                                               │
│  1. 解析 CLI 参数 (host, port, project_path, instance_type...) │
│  2. project_path = os.path.abspath(project_path)               │
│  3. set_repo_path(project_path)                                │
│     → os.environ['MAGE_REPO_PATH'] = project_path             │
│     → sys.path.append(dirname(project_path))                   │
│     → set_project_platform_activated_flag()                    │
│  4. initialize_globals()                                       │
│     → get_job_manager() 打印初始化                             │
│     → KERNEL_MAGIC? get_execution_result_queue()               │
│  5. start_server(...)                                          │
└───────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────┐
│ Phase 2: start_server() (server/server.py)                     │
│                                                               │
│  1. 确定 project 路径，若不存在则 init_repo() 初始化            │
│  2. set_repo_path(project) 再次确认                             │
│  3. init_project_uuid(overwrite_uuid, root_project=True)       │
│  4. asyncio.run(UsageStatisticLogger().project_impression())   │
│  5. set_logging_format(logging_format, level)                  │
│  6. ┌─ manage 或 ProjectType.MAIN?                            │
│     │   → os.environ[MAGE_IS_MANAGE_INSTANCE] = '1'           │
│     │   → database_manager.run_migrations()                    │
│     ├─ InstanceType.SERVER_AND_SCHEDULER?                      │
│     │   → scheduler_manager.start_scheduler()                  │
│     ├─ InstanceType.SCHEDULER?                                 │
│     │   → scheduler_manager.start_scheduler()                  │
│     │   → run_web_server_with_status_only = True               │
│     └─ InstanceType.WEB_SERVER?                                │
│         → database_manager.run_migrations()                    │
│  7. asyncio.run(main(...))                                     │
└───────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────┐
│ Phase 3: main() 异步主函数 (server/server.py)                   │
│                                                               │
│  1. switch_active_kernel(DEFAULT_KERNEL_NAME)                  │
│  2. BASE_PATH 处理 → replace_base_path() 替换前端静态文件      │
│  3. make_app() 构建 Tornado Application                        │
│     → 路由注册 (API v1, WebSocket, Static, Pages)             │
│     → Prometheus/OTEL 插桩（条件启用）                          │
│     → ROUTES_BASE_PATH 路由前缀处理                            │
│  4. app.listen(port)                                           │
│  5. db_connection.start_session(force=True)                    │
│  6. latest_user_activity.update_latest_activity()              │
│  7. Git 同步（如果 preferences.sync_config 存在）               │
│  8. ┌─ REQUIRE_USER_AUTHENTICATION?                            │
│     │   → initialize_user_authentication()                     │
│     │     → Role.create_default_roles()                        │
│     │     → 创建默认 owner 用户                                 │
│     │     → 创建 OAuth2 Application                            │
│     └─ REQUIRE_USER_PERMISSIONS? 打印日志                      │
│  9. BlockCache.initialize_cache() + PipelineCache              │
│ 10. TagCache.initialize_cache()                                │
│ 11. BlockActionObjectCache.initialize_cache()                  │
│ 12. ┌─ FeatureUUID.COMPUTE_MANAGEMENT + spark_config?          │
│     │   → Application.clear_cache()                            │
│     ├─ FeatureUUID.DBT_V2?                                     │
│     │   → DBTCache.initialize_cache_async()                    │
│     └─ FeatureUUID.COMMAND_CENTER?                             │
│         → FileCache.initialize_cache_with_settings()           │
│ 13. SSH 隧道创建（EMR 场景）                                   │
│ 14. ┌─ ProjectType.MAIN?                                       │
│     │   → PeriodicCallback(check_auto_termination, 60s)        │
│     └─ else:                                                   │
│         → PeriodicCallback(check_scheduler_status, 20s)         │
│ 15. get_memory_manager_controller().events                     │
│ 16. update_settings_on_metadata_change()                       │
│ 17. Observer 监听 metadata.yaml 文件变更                       │
│ 18. get_messages() → WebSocketServer 消息管道                  │
│ 19. await asyncio.Event().wait()  ← 阻塞主线程                │
└───────────────────────────────────────────────────────────────┘
```

---

## 七、数据库连接初始化

[orchestration/db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/orchestration/db/__init__.py) 在 **模块导入时** 即完成数据库引擎创建：

```
1. os.getenv('MAGE_DATABASE_CONNECTION_URL')
2. is_test_mage()? → sqlite:///test.db
3. get_postgres_connection_url()  ← 来自 PG_DB_* 环境变量或 AWS/Azure Secrets
4. 兜底: sqlite:///$variables_dir/mage-ai.db
5. PostgreSQL? → pool_size=50, timezone=utc
6. OTEL 启用? → SQLAlchemyInstrumentor().instrument()
7. create_engine(db_connection_url, **db_kwargs)
8. session_factory = sessionmaker(class_=SessionWithCaching, bind=engine)
```

关键判断：PostgreSQL 连接池大小 50，SQLite 使用 `check_same_thread=False`。

---

## 八、RepoConfig 与 metadata.yaml

[repo_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/repo_manager.py#L45-L176) 的 `RepoConfig` 类从项目根目录的 `metadata.yaml` 加载运行配置：

| 字段 | 说明 |
|------|------|
| `project_type` | main / sub / standalone |
| `project_uuid` | 项目唯一标识 |
| `cluster_type` | emr / ecs / cloud_run / k8s |
| `variables_dir` | 变量存储路径（本地/远程） |
| `remote_variables_dir` | S3/GCS 远程变量路径 |
| `ecs_config` | AWS ECS 执行器配置 |
| `emr_config` | AWS EMR 执行器配置 |
| `gcp_cloud_run_config` | GCP Cloud Run 配置 |
| `k8s_executor_config` | K8s 执行器配置 |
| `spark_config` | Spark 执行器配置 |
| `notification_config` | 通知配置（Slack/Email 等） |
| `ai_config` | AI 功能配置 |
| `features` | Feature Flag 字典 |
| `ldap_config` | LDAP 认证配置 |
| `settings_backend` | 设置后端配置（AWS Secrets Manager） |
| `logging_config` | 日志配置 |
| `queue_config` | 队列配置 |
| `retry_config` | 重试配置 |
| `variables_retention_period` | 变量保留期 |

metadata.yaml 支持 Jinja2 模板渲染，可通过 `{{ env_var('KEY') }}` 引用环境变量。

---

## 九、执行器工厂与 Pipeline 运行参数

[executor_factory.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/data_preparation/executors/executor_factory.py) 根据多重条件选择执行器：

### Pipeline 执行器选择

```
1. 显式指定 executor_type → 直接使用
2. pipeline.type == PYSPARK → PySparkPipelineExecutor
3. pipeline.get_executor_type() != LOCAL_PYTHON → 使用 pipeline 配置
4. os.getenv('DEFAULT_EXECUTOR_TYPE') → 默认执行器类型
5. 兜底 → PipelineExecutor (本地)
```

### Block 执行器选择

```
1. 显式指定 executor_type → 直接使用
2. pipeline.type == PYSPARK 且 block 不是 SENSOR / 含 spark 代码 → PySparkBlockExecutor
3. block.get_executor_type() != LOCAL_PYTHON → 使用 block 配置
4. os.getenv('DEFAULT_EXECUTOR_TYPE') → 默认执行器类型
5. 兜底 → BlockExecutor (本地)
```

支持的执行器类型：LOCAL_PYTHON, PYSPARK, ECS, K8S, GCP_CLOUD_RUN, AZURE_CONTAINER_INSTANCE。

---

## 十、环境判断工具函数

[environments.py](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/shared/environments.py) 提供了全局环境判断：

| 函数 | 判断依据 | 用途 |
|------|---------|------|
| `is_test_mage()` | `ENV=test_mage` 或 `unittest` in sys.argv | CI/CD 单元测试 |
| `is_test()` | `ENV=test` 或 is_test_mage() | 用户测试环境 |
| `is_dev()` | `ENV=dev` 或 `ENV=development` | 开发环境（启用 autoreload） |
| `is_production()` | `ENV=production` 或 `ENV=prod` | 生产环境 |
| `is_debug()` | `DEBUG=1` | 调试模式（异常时 raise） |
| `is_deus_ex_machina()` | `DEUS_EX_MACHINA=1` | 神秘模式 |

这些判断在多处关键分支中使用：
- `is_test_mage()` → 决定数据库使用 `test.db`
- `is_dev()` → 决定是否启用 Tornado autoreload
- `is_debug()` → 决定异常时是否 raise 而非仅 print

---

## 十一、关键判断节点汇总

| 判断点 | 所在文件 | 条件 | 影响 |
|--------|---------|------|------|
| 实例类型 | server.py start_server() | `INSTANCE_TYPE` | 决定是否启动调度器/仅 Web/仅调度 |
| 项目类型 | server.py start_server() | `ProjectType` | 决定是否设 MANAGE_ENV_VAR/执行迁移 |
| 用户认证 | server.py main() | `REQUIRE_USER_AUTHENTICATION` | 创建角色/用户/OAuth 应用 |
| 权限控制 | server.py main() | `REQUIRE_USER_PERMISSIONS` | 启用 RBAC |
| 认证模式 | server.py initialize_user_authentication() | `AUTHENTICATION_MODE` | LDAP 或本地密码创建用户 |
| 多项目平台 | repo.py get_repo_path() | `project_platform_activated()` | 活跃项目路径 vs 根项目路径 |
| 变量存储 | repo.py get_variables_dir() | `MAGE_DATA_DIR` / metadata.yaml | 本地/ S3 / GCS |
| 数据库类型 | orchestration/db/__init__.py | `MAGE_DATABASE_CONNECTION_URL` | PostgreSQL vs SQLite |
| 执行器类型 | executor_factory.py | `DEFAULT_EXECUTOR_TYPE` / pipeline 配置 | 本地/ ECS / K8s / PySpark |
| Base Path | server.py make_app() | `MAGE_BASE_PATH` | 路由前缀 + 前端静态文件替换 |
| 可观测性 | server.py make_app() | `ENABLE_PROMETHEUS` / `OTEL_*` | Prometheus 指标端点 / OTEL 插桩 |
| Git 同步 | server.py main() | `preferences.sync_config` | 启动时拉取远程仓库 |
| Feature Flag | server.py main() | `FeatureUUID.*` | DBT_V2 / COMMAND_CENTER / COMPUTE_MANAGEMENT |
| Scheduler 重启 | scheduler_manager.py | `SCHEDULER_AUTO_RESTART_INTERVAL` (20s) | 自动恢复调度器进程 |
| Kernel 类型 | setup.py initialize_globals() | `KERNEL_MANAGER=magic` | 初始化全局执行结果队列 |

---

## 十二、全局单例对象

| 单例 | 所在文件 | 创建时机 | 作用 |
|------|---------|---------|------|
| `settings` | settings/__init__.py | 模块导入时 | 统一设置读取入口 |
| `scheduler_manager` | server/scheduler_manager.py | 模块导入时 | 调度器进程生命周期管理 |
| `latest_user_activity` | server/server.py | 模块导入时 | 活跃时间追踪（支持 Redis） |
| `engine` | orchestration/db/__init__.py | 模块导入时 | SQLAlchemy 数据库引擎 |
| `db_connection` | orchestration/db/__init__.py | 模块导入时 | 数据库会话管理 |
| `MemoryManagerController` | shared/singletons/memory.py | 首次调用时 | 内存管理事件监控（@singleton 装饰器） |

---

## 十三、`mage run` CLI 命令的参数流

[cli/main.py run](file:///d:/fz/0601/solo-dogfeeding/code/324-mage-ai/mage_ai/cli/main.py#L160-L276) 的运行参数流：

```
CLI 参数:
  project_path, pipeline_uuid, block_uuid,
  test, execution_partition, executor_type,
  callback_url, block_run_id, pipeline_run_id,
  runtime_vars, skip_sensors, template_runtime_configuration

                    ↓

1. set_repo_path(project_path)     ← 确定仓库路径
2. Sentry / New Relic 初始化        ← 可观测性
3. Git 同步（条件: sync_on_executor_start）
4. parse_runtime_variables(runtime_vars)  ← 解析运行时变量
5. db_connection.start_session()    ← 数据库会话
6. Pipeline.get(pipeline_uuid)      ← 加载 Pipeline 模型

                    ↓

全局变量合并:
  ┌─ pipeline_run_id is None?
  │   → get_global_variables(pipeline_uuid)  ← 项目全局变量
  │   → merge_dict(default_variables, runtime_variables)
  └─ pipeline_run_id exists?
      → pipeline_run.get_variables(extra_variables=runtime_variables)

                    ↓

执行:
  ┌─ block_uuid is None?  → ExecutorFactory.get_pipeline_executor(...).execute(...)
  └─ block_uuid exists?  → ExecutorFactory.get_block_executor(...).execute(...)
```

---

## 十四、配置优先级全景

从最高到最低：

```
1. CLI 显式参数 (host, port, project_path...)
2. 环境变量 (MAGE_REPO_PATH, INSTANCE_TYPE, MAGE_DATABASE_CONNECTION_URL...)
3. Settings 后端存储 (AWS Secrets Manager)
4. metadata.yaml (Jinja2 渲染后的值)
5. 项目偏好文件 (preferences.yaml / sync_config)
6. io_config.yaml (IO 配置 profile)
7. 代码硬编码默认值 (DEFAULT_LOCALHOST_URL, DATA_PREP_SERVER_PORT=6789...)
```

特殊优先级规则：
- Git 配置：环境变量 > 项目偏好，除非 `GIT_OVERWRITE_WITH_PROJECT_SETTINGS=True`
- 变量目录：`MAGE_DATA_DIR` 环境变量 > metadata.yaml > 默认 `~/.mage_data`
- 数据库 URL：`MAGE_DATABASE_CONNECTION_URL` > PG 环境变量组合 > AWS/Azure Secrets > SQLite

# Trigger 与 Schedule 代码实现分析

## 一、核心概念与三层架构

Trigger（触发器）与 Schedule（调度）是 Mage AI 中管道执行的两套核心机制，通过三层架构协作：

```
配置层: Trigger (triggers.yaml)
     ↓ sync_schedules()
调度层: PipelineSchedule (数据库)
     ↓ schedule_all() / schedule_with_event() / API
执行层: PipelineRun → BlockRun
```

| 层级 | 概念 | 存储 | 核心职责 |
|------|------|------|----------|
| 配置层 | **Trigger** | 每个管道目录下的 `triggers.yaml` | 定义"什么时候以什么方式触发管道执行" |
| 调度层 | **PipelineSchedule** | 数据库 `pipeline_schedule` 表 | Trigger 的运行时持久化表示，负责调度决策 |
| 执行层 | **PipelineRun / BlockRun** | 数据库 `pipeline_run` / `block_run` 表 | 每次触发产生的具体运行实例 |

**核心桥梁**：`sync_schedules()` 函数负责将代码中的 Trigger 配置同步到数据库的 PipelineSchedule 中。

---

## 二、核心对象模型与代码位置

### 2.1 Trigger（触发器配置）

**文件**：[`mage_ai/data_preparation/models/triggers/__init__.py`](./mage_ai/data_preparation/models/triggers/__init__.py)

Trigger 是基于 dataclass 的配置对象，定义在 `triggers.yaml` 文件中，描述管道的触发规则。

| 类/枚举 | 起始行 | 说明 |
|-----------|--------|------|
| `ScheduleStatus` | [L22-L25](./mage_ai/data_preparation/models/triggers/__init__.py#L22-L25) | 状态枚举：`ACTIVE` / `INACTIVE` |
| `ScheduleType` | [L27-L31](./mage_ai/data_preparation/models/triggers/__init__.py#L27-L31) | 类型枚举：`TIME` / `EVENT` / `API` |
| `ScheduleInterval` | [L40-L47](./mage_ai/data_preparation/models/triggers/__init__.py#L40-L47) | 间隔枚举：`@once` / `@hourly` / `@daily` / `@weekly` / `@monthly` / `@always_on` |
| `SettingsConfig` | [L50-L58](./mage_ai/data_preparation/models/triggers/__init__.py#L50-L58) | 高级设置（并发控制、landing time 等） |
| `Trigger` | [L61-L117](./mage_ai/data_preparation/models/triggers/__init__.py#L61-L117) | Trigger 配置主类 |

**Trigger 核心字段**：
- `name`：触发器名称，与 `pipeline_uuid` 联合唯一
- `schedule_type` / `schedule_interval`：触发类型与间隔
- `start_time` / `last_enabled_at`：时间边界
- `status`：启用状态
- `variables` / `settings` / `sla`：运行配置

**SettingsConfig 字段**：
- `skip_if_previous_running`：前次运行未完成时跳过
- `allow_blocks_to_fail`：允许部分 Block 失败
- `create_initial_pipeline_run`：启用时立即创建初始运行
- `landing_time_enabled`：启用落地时间机制
- `pipeline_run_limit`：单触发器并发运行数限制
- `timeout` / `timeout_status`：超时配置

**主要工具函数**：

| 函数 | 起始行 | 说明 |
|------|--------|------|
| `get_triggers_by_pipeline()` | [L232-L238](./mage_ai/data_preparation/models/triggers/__init__.py#L232-L238) | 获取管道的所有触发器 |
| `load_trigger_configs()` | [L285-L300](./mage_ai/data_preparation/models/triggers/__init__.py#L285-L300) | 从 yaml 内容加载触发器配置 |
| `add_or_update_trigger_for_pipeline_and_persist()` | [L303-L329](./mage_ai/data_preparation/models/triggers/__init__.py#L303-L329) | 添加/更新触发器并持久化到文件 |

### 2.2 PipelineSchedule（调度任务）

**文件**：[`mage_ai/orchestration/db/models/schedules.py`](./mage_ai/orchestration/db/models/schedules.py)

PipelineSchedule 是数据库模型，对应 `pipeline_schedule` 表，是 Trigger 的运行时持久化形态。

**类起始行**：[L86-L766](./mage_ai/orchestration/db/models/schedules.py#L86-L766)

**核心方法**：

| 方法 | 起始行 | 说明 |
|------|--------|------|
| `active_schedules()` | [L293-L299](./mage_ai/orchestration/db/models/schedules.py#L293-L299) | 类方法，获取所有活跃调度 |
| `create()` | [L302-L306](./mage_ai/orchestration/db/models/schedules.py#L302-L306) | 类方法，创建调度 |
| `create_or_update_batch()` | [L309-L407](./mage_ai/orchestration/db/models/schedules.py#L309-L407) | 批量创建/更新（`sync_schedules` 调用） |
| `current_execution_date()` | [L438-L508](./mage_ai/orchestration/db/models/schedules.py#L438-L508) | 计算当前执行日期 |
| `next_execution_date()` | [L510-L534](./mage_ai/orchestration/db/models/schedules.py#L510-L534) | 计算下次执行日期 |
| `should_schedule()` | [L537-L679](./mage_ai/orchestration/db/models/schedules.py#L537-L679) | **核心决策方法**，判断是否应该调度新运行 |
| `landing_time_enabled()` | [L681-L693](./mage_ai/orchestration/db/models/schedules.py#L681-L693) | 是否启用落地时间 |
| `runtime_history()` | [L725-L750](./mage_ai/orchestration/db/models/schedules.py#L725-L750) | 获取历史运行时长 |

**`should_schedule()` 判断逻辑**：

```
状态检查 (ACTIVE?)
   ↓
start_time 检查（非 landing_time 模式下）
   ↓
类型分支判断
   ├─ @once: 运行数为0 → True；流式且executor_count>1 → 可多实例
   ├─ @always_on: 无运行 → True；非运行/非初始 → True
   └─ 其他时间间隔:
        ├─ 计算 current_execution_date
        ├─ last_enabled_at 边界检查（避免启用前的历史运行）
        ├─ start_time 检查
        ├─ 同 execution_date 的运行是否已存在（去重）
        └─ landing_time 额外判断（基于历史运行时长预测）
```

### 2.3 PipelineRun（管道运行实例）

**文件**：[`mage_ai/orchestration/db/models/schedules.py`](./mage_ai/orchestration/db/models/schedules.py)

**类起始行**：[L768-L1388](./mage_ai/orchestration/db/models/schedules.py#L768-L1388)

**状态枚举 `PipelineRunStatus`**：[L769-L774](./mage_ai/orchestration/db/models/schedules.py#L769-L774)
- `INITIAL`：初始状态，已创建未开始
- `RUNNING`：运行中
- `COMPLETED`：已完成
- `FAILED`：失败
- `CANCELLED`：已取消

**核心方法**：

| 方法 | 起始行 | 说明 |
|------|--------|------|
| `create()` | [L1353-L1388](./mage_ai/orchestration/db/models/schedules.py#L1353-L1388) | 创建运行，含 `prevent_duplicates` 去重参数 |
| `executable_block_runs()` | [L978-L1221](./mage_ai/orchestration/db/models/schedules.py#L978-L1221) | 获取当前可执行的 Block 列表 |
| `update_block_run_statuses()` | [L1223-L1301](./mage_ai/orchestration/db/models/schedules.py#L1223-L1301) | 级联更新 Block 运行状态 |
| `create_block_runs()` | [L1443](./mage_ai/orchestration/db/models/schedules.py#L1443) | 创建所有 BlockRun |

**去重逻辑**（`prevent_duplicates` 参数）：
去重键为三元组 `(execution_date, pipeline_schedule_id, pipeline_uuid)`，存在相同三元组则返回 `None`，不创建。

### 2.4 BlockRun（代码块运行实例）

**文件**：[`mage_ai/orchestration/db/models/schedules.py`](./mage_ai/orchestration/db/models/schedules.py)

**类起始行**：[L1663-L1922](./mage_ai/orchestration/db/models/schedules.py#L1663-L1922)

**状态枚举 `BlockRunStatus`**：[L1664-L1672](./mage_ai/orchestration/db/models/schedules.py#L1664-L1672)
- `INITIAL` → `QUEUED` → `RUNNING` → `COMPLETED` / `FAILED` / `CANCELLED` / `UPSTREAM_FAILED` / `CONDITION_FAILED`

### 2.5 EventMatcher（事件匹配器）

**文件**：[`mage_ai/orchestration/db/models/schedules.py`](./mage_ai/orchestration/db/models/schedules.py)

**类起始行**：[L1924-L2023](./mage_ai/orchestration/db/models/schedules.py#L1924-L2023)

用于事件触发模式，通过 JSON 模式递归匹配事件内容。

| 方法 | 起始行 | 说明 |
|------|--------|------|
| `match()` | [L2007-L2022](./mage_ai/orchestration/db/models/schedules.py#L2007-L2022) | 递归匹配事件字典与 pattern |
| `active_event_matchers()` | [L1942-L1945](./mage_ai/orchestration/db/models/schedules.py#L1942-L1945) | 获取所有活跃的事件匹配器 |

---

## 三、API 触发完整流程

### 3.1 API 触发入口

API 触发有两个独立入口点：

#### 入口一：完整管道触发

**文件**：[`mage_ai/server/api/triggers.py`](./mage_ai/server/api/triggers.py)

**端点**：`POST /api/pipeline_schedules/{pipeline_schedule_id}/triggers`

**处理类**：`ApiTriggerPipelineHandler`，`post()` 方法 [L18-L65](./mage_ai/server/api/triggers.py#L18-L65)

**执行流程**：

```
HTTP 请求到达
   ↓
Token 验证 (L35-L40)
   ├─ 检查 schedule_type == API
   ├─ 检查 token 非空
   └─ 验证 token 匹配
   ↓
构建 payload (L42-L53)
   ├─ variables: 从 URL 查询参数解析
   └─ event_variables: 从请求体 JSON 解析
   ↓
调用 create_and_start_pipeline_run(
   pipeline,
   pipeline_schedule,
   payload,
   should_schedule=False   ← 关键参数！
)
   ↓
返回 PipelineRun 字典
```

**关键设计**：`should_schedule=False` —— API 触发**不立即启动**调度器，只创建 `INITIAL` 状态的 PipelineRun，交由定时调度器统一处理。

#### 入口二：单个 Block 运行触发

**文件**：[`mage_ai/server/api/runs.py`](./mage_ai/server/api/runs.py)

**端点**：`POST /api/runs`

**处理类**：`ApiRunHandler`，`post()` 方法 [L27-L150](./mage_ai/server/api/runs.py#L27-L150)

这个入口用于直接运行单个 Block，不经过标准调度流程，同步执行并返回结果。

### 3.2 创建运行：create_and_start_pipeline_run()

**文件**：[`mage_ai/orchestration/triggers/utils.py`](./mage_ai/orchestration/triggers/utils.py)

**函数起始行**：[L83-L136](./mage_ai/orchestration/triggers/utils.py#L83-L136)

**函数签名**：

```python
def create_and_start_pipeline_run(
    pipeline: Pipeline,
    pipeline_schedule: PipelineSchedule,
    payload: Dict = None,
    should_schedule: bool = False,  # API 触发传入 False
    remote_blocks: List = None,
) -> PipelineRun:
```

**执行步骤**：

1. **配置 payload**（[L93-L97](./mage_ai/orchestration/triggers/utils.py#L93-L97)）
   - 调用 `configure_pipeline_run_payload()` 标准化
   - 设置 `pipeline_schedule_id`、`pipeline_uuid`、`execution_date`
   - 注入 `execution_partition` 到 variables

2. **创建 PipelineRun**（[L106](./mage_ai/orchestration/triggers/utils.py#L106)）

   ```python
   pipeline_run = PipelineRun.create(**configured_payload)
   ```

   - 未使用 `prevent_duplicates`，每次调用都创建
   - 创建后状态为 `INITIAL`

3. **条件性启动**（[L108-L131](./mage_ai/orchestration/triggers/utils.py#L108-L131)）

   ```python
   if should_schedule:  # API 触发时为 False，跳过
       pipeline_scheduler = PipelineScheduler(pipeline_run)
       pipeline_scheduler.start(should_schedule=should_schedule)
   ```

4. **激活 Schedule**（[L133-L134](./mage_ai/orchestration/triggers/utils.py#L133-L134)）
   - PipelineSchedule 若非 `ACTIVE` 状态，自动激活

> **设计意图**：
> API 触发故意不立即启动调度器，将 PipelineRun 留在 `INITIAL` 状态，
> 等待定时调度器 `schedule_all()` 统一处理。好处：
> 1. 所有触发方式（时间/事件/API）都经过相同的并发控制
> 2. 避免 API 请求线程执行重量级调度操作

---

## 四、调度过程详解

### 4.1 主调度器：schedule_all()

**文件**：[`mage_ai/orchestration/pipeline_scheduler_original.py`](./mage_ai/orchestration/pipeline_scheduler_original.py)

**函数起始行**：[L1656-L1955](./mage_ai/orchestration/pipeline_scheduler_original.py#L1656-L1955)

由 `LoopTimeTrigger` 定时循环调用，是整个调度系统的心脏。

**完整流程**：

```
schedule_all() 被调用
   ↓
1. sync_schedules() 同步 yaml → DB [L1685-L1688]
   ↓
2. 获取所有 ACTIVE 状态的 PipelineSchedule [L1690-L1694]
   ↓
3. 按 pipeline_uuid 分组处理
   ↓
4. 对每个 PipelineSchedule:
   ├─ 获取分布式锁 pipeline_schedule_{id} [L1755-L1757]
   ├─ should_schedule() 判断是否需要新运行 [L1788-L1795]
   ├─ 获取 initial_pipeline_runs（INITIAL 状态的运行）[L1796]
   │  ↑ API 触发创建的运行就在这里！
   │
   ├─ 计算单触发器并发配额 [L1759-L1775]
   │  ├─ trigger_pipeline_run_limit
   │  └─ running_pipeline_run_count
   │
   ├─ 按 execution_date 排序 [L1881]
   ├─ 按配额选择要启动的运行 [L1880-L1885]
   └─ 超限运行加入 excluded 列表
   ↓
5. 管道级并发限制（全触发器）[L1889-L1906]
   ├─ pipeline_run_limit_all_triggers
   └─ 再次过滤
   ↓
6. 启动选中的运行 [L1908-L1915]
   └─ PipelineScheduler(r).start()
   ↓
7. 处理超限运行 [L1917-L1926]
   ├─ WAIT 策略：保留 INITIAL，排队等待
   └─ SKIP 策略：设置为 CANCELLED
   ↓
8. 处理所有 RUNNING 状态的运行 [L1928-L1942]
   └─ PipelineScheduler(r).schedule()
```

**关键代码位置**：
- 初始运行获取：[L1796](./mage_ai/orchestration/pipeline_scheduler_original.py#L1796)
- 单触发器并发限制：[L1872-L1885](./mage_ai/orchestration/pipeline_scheduler_original.py#L1872-L1885)
- 管道全触发器并发限制：[L1889-L1906](./mage_ai/orchestration/pipeline_scheduler_original.py#L1889-L1906)
- 启动 PipelineRun：[L1908-L1915](./mage_ai/orchestration/pipeline_scheduler_original.py#L1908-L1915)
- 超限策略处理：[L1917-L1926](./mage_ai/orchestration/pipeline_scheduler_original.py#L1917-L1926)

### 4.2 PipelineScheduler 类

**文件**：[`mage_ai/orchestration/pipeline_scheduler_original.py`](./mage_ai/orchestration/pipeline_scheduler_original.py)

**类起始行**：[L78](./mage_ai/orchestration/pipeline_scheduler_original.py#L78)

负责单个 PipelineRun 的生命周期管理。

**核心方法**：

| 方法 | 起始行 | 说明 |
|------|--------|------|
| `start()` | [L131-L203](./mage_ai/orchestration/pipeline_scheduler_original.py#L131-L203) | 启动管道运行 |
| `stop()` | [L206-L210](./mage_ai/orchestration/pipeline_scheduler_original.py#L206-L210) | 停止管道运行 |
| `schedule()` | [L213](./mage_ai/orchestration/pipeline_scheduler_original.py#L213) | 调度可执行 Block |
| `on_block_complete()` | [L377-L410](./mage_ai/orchestration/pipeline_scheduler_original.py#L377-L410) | Block 完成回调 |
| `on_block_failure()` | [L443-L489](./mage_ai/orchestration/pipeline_scheduler_original.py#L443-L489) | Block 失败回调 |

#### `start()` 方法流程：

```
获取分布式锁 pipeline_run_{id} [L148-L150]
   ↓
检查 block_runs 是否已创建 [L157-L175]
   └─ 未创建则调用 create_block_runs()
   ↓
更新状态：INITIAL → RUNNING [L197-L200]
   ↓
如果 should_schedule=True，调用 self.schedule() [L201-L202]
   ↓
释放锁 [L195]
```

**`should_schedule` 参数含义**：
- `True`：启动后立即调用 `schedule()` 推进 Block 执行（时间触发使用）
- `False`：仅更新状态为 RUNNING，不立即调度（特殊场景使用）

### 4.3 事件触发：schedule_with_event()

**文件**：[`mage_ai/orchestration/pipeline_scheduler_original.py`](./mage_ai/orchestration/pipeline_scheduler_original.py)

**函数起始行**：[L2067-L2108](./mage_ai/orchestration/pipeline_scheduler_original.py#L2067-L2108)

**流程**：
1. 获取所有活跃 EventMatcher
2. 遍历匹配器，调用 `e.match(event)` 模式匹配
3. 对匹配成功的调度，创建 PipelineRun
4. 事件数据通过 `variables.event` 传递给管道

### 4.4 配置同步：sync_schedules()

**文件**：[`mage_ai/orchestration/pipeline_scheduler_original.py`](./mage_ai/orchestration/pipeline_scheduler_original.py)

**函数起始行**：[L2110-L2129](./mage_ai/orchestration/pipeline_scheduler_original.py#L2110-L2129)

负责将 `triggers.yaml` 中的 Trigger 配置同步到数据库 PipelineSchedule：
1. 遍历所有管道，读取 `triggers.yaml`
2. 按环境过滤（`envs` 字段）
3. 调用 `PipelineSchedule.create_or_update_batch()` 批量同步

### 4.5 SLA 检查：check_sla()

**函数起始行**：[L1606-L1654](./mage_ai/orchestration/pipeline_scheduler_original.py#L1606-L1654)

遍历进行中的 PipelineRun，比较 `execution_date + sla` 与当前时间，超时则标记 `passed_sla=True` 并发送通知。

---

## 五、并发去重机制

### 5.1 去重机制：prevent_duplicates

**位置**：`PipelineRun.create()` 方法，[L1374-L1381](./mage_ai/orchestration/db/models/schedules.py#L1374-L1381)

**去重键**：三元组 `(execution_date, pipeline_schedule_id, pipeline_uuid)`

```python
if prevent_duplicates:
    existing_pipeline_run = PipelineRun.query.filter(
        PipelineRun.execution_date == kwargs.get('execution_date'),
        PipelineRun.pipeline_schedule_id == kwargs.get('pipeline_schedule_id'),
        PipelineRun.pipeline_uuid == kwargs.get('pipeline_uuid'),
    ).first()
    if existing_pipeline_run is not None:
        return None
```

**各触发方式的去重策略**：

| 触发方式 | prevent_duplicates | 原因 |
|----------|-------------------|------|
| 时间触发 | **True** | 避免同一调度周期重复创建 |
| 事件触发 | **False** | 每次事件独立 |
| API 触发 | **False** | 每次 API 调用独立，`execution_date` 每次不同 |

### 5.2 并发控制：三级限制体系

**配置文件**：[`mage_ai/orchestration/concurrency.py`](./mage_ai/orchestration/concurrency.py)

**ConcurrencyConfig**：

```python
@dataclass
class ConcurrencyConfig(BaseConfig):
    block_run_limit: int = ...                    # Block 级
    pipeline_run_limit: int = ...                 # 单触发器级
    pipeline_run_limit_all_triggers: int = None   # 管道全触发器级
    on_pipeline_run_limit_reached: str = OnLimitReached.WAIT
```

#### 第一级：单触发器并发限制

**代码位置**：[L1759-L1885](./mage_ai/orchestration/pipeline_scheduler_original.py#L1759-L1885)

```
trigger_pipeline_run_limit = settings.pipeline_run_limit 或 concurrency_config.pipeline_run_limit
running_count = pipeline_schedule.running_pipeline_run_count
quota = trigger_pipeline_run_limit - running_count
```

优先级：按 `execution_date` 升序，先创建的先启动。

#### 第二级：管道全触发器并发限制

**代码位置**：[L1889-L1906](./mage_ai/orchestration/pipeline_scheduler_original.py#L1889-L1906)

限制一个管道所有触发器的总并发数。

#### 第三级：Block 级并发限制

在 `PipelineScheduler.schedule()` 中控制单管道内并发 Block 数。

### 5.3 超限策略：WAIT vs SKIP

**代码位置**：[L1917-L1926](./mage_ai/orchestration/pipeline_scheduler_original.py#L1917-L1926)

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| **WAIT**（默认） | 保留 `INITIAL` 状态，下次 `schedule_all()` 重试 | 周期性任务，延迟可接受 |
| **SKIP** | 设置为 `CANCELLED`，记录日志后丢弃 | 实时性要求高，错过无意义 |

### 5.4 分布式锁

**文件**：[`mage_ai/orchestration/utils/distributed_lock.py`](./mage_ai/orchestration/utils/distributed_lock.py)

调度过程中使用的锁：

| 锁 Key | 保护范围 | 位置 |
|--------|----------|------|
| `pipeline_schedule_{id}` | 单个调度的决策过程 | [L1755-L1757](./mage_ai/orchestration/pipeline_scheduler_original.py#L1755-L1757) |
| `pipeline_run_{id}` | 单个运行的启动/调度过程 | [L148-L150](./mage_ai/orchestration/pipeline_scheduler_original.py#L148-L150) |

作用：防止多实例部署时重复调度、重复启动。

---

## 六、API 触发 → 创建运行 → 启动调度 → 并发去重 完整关联

```
HTTP API 请求
   │
   ▼
┌──────────────────────────────────────────┐
│ ApiTriggerPipelineHandler.post()        │
│ mage_ai/server/api/triggers.py          │
│  ├─ Token 验证                          │
│  ├─ 构建 payload (variables, event)     │
│  └─ create_and_start_pipeline_run()     │
│     should_schedule=False               │
└─────────────────────┬────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────┐
│ create_and_start_pipeline_run()          │
│ mage_ai/orchestration/triggers/utils.py  │
│  ├─ configure_pipeline_run_payload()     │
│  ├─ PipelineRun.create()                 │
│  │  └─ prevent_duplicates=False          │
│  │     → 状态: INITIAL                   │
│  ├─ should_schedule=False → 不启动       │
│  └─ 激活 PipelineSchedule (如未激活)     │
└─────────────────────┬────────────────────┘
                      │
                      │ 时间流逝...
                      │
                      ▼
┌──────────────────────────────────────────┐
│ schedule_all()  [定时轮询]               │
│ mage_ai/orchestration/                   │
│   pipeline_scheduler_original.py         │
│                                          │
│ 对每个 PipelineSchedule:                 │
│  ├─ 锁: pipeline_schedule_{id}           │
│  ├─ should_schedule() 时间触发判断       │
│  ├─ initial_pipeline_runs ← API 触发的   │
│  ├─ 计算并发配额:                        │
│  │  ├─ trigger_pipeline_run_limit        │
│  │  ├─ pipeline_run_limit_all_triggers   │
│  │  └─ running_pipeline_run_count        │
│  ├─ 按配额筛选 + 排序                    │
│  └─ 超限处理: WAIT / SKIP                │
│                                          │
│ 启动选中的运行:                           │
│  └─ PipelineScheduler(r).start()         │
│     ├─ 创建 BlockRuns                    │
│     └─ 状态: INITIAL → RUNNING           │
│                                          │
│ 推进 RUNNING 状态的运行:                 │
│  └─ PipelineScheduler(r).schedule()      │
│     ├─ 检查超时                          │
│     ├─ 调度可执行 Blocks                 │
│     └─ 更新状态                          │
└──────────────────────────────────────────┘
```

---

## 七、边界条件与特殊机制

### 7.1 skip_if_previous_running

**代码位置**：[L1835-L1852](./mage_ai/orchestration/pipeline_scheduler_original.py#L1835-L1852)

如果前一次运行还在进行中，不创建新运行，而是创建一个 `CANCELLED` 占位记录。

### 7.2 last_enabled_at 边界

**代码位置**：[L614-L640](./mage_ai/orchestration/db/models/schedules.py#L614-L640)

避免调度器启用时追溯历史执行日期。

### 7.3 Landing Time 机制

**代码位置**：[L660-L676](./mage_ai/orchestration/db/models/schedules.py#L660-L676)

确保管道在指定时间点前完成：
1. 收集最近 7 次历史运行时长
2. 计算 `平均运行时间 + 标准差/2` 作为缓冲
3. 当前时间 ≥ 预期执行时间 − (平均 + 缓冲) → 开始调度

**注意**：启用 Landing Time 后，`start_time` 的语义从"开始时间"变为"完成截止时间"。

### 7.4 两级超时

| 级别 | 配置位置 | 检查时机 |
|------|----------|----------|
| 管道超时 | `PipelineSchedule.settings.timeout` | 每次 `schedule()` 调用 |
| Block 超时 | `Block.timeout` | Block 调度循环中 |

超时后状态默认为 `FAILED`，可通过 `timeout_status` 自定义。

---

## 八、关键代码索引

### 8.1 核心文件

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| Trigger 配置 | [`mage_ai/data_preparation/models/triggers/__init__.py`](./mage_ai/data_preparation/models/triggers/__init__.py) | Trigger 数据类、yaml 读写 |
| 调度模型 | [`mage_ai/orchestration/db/models/schedules.py`](./mage_ai/orchestration/db/models/schedules.py) | PipelineSchedule / PipelineRun / BlockRun / EventMatcher |
| 主调度器 | [`mage_ai/orchestration/pipeline_scheduler_original.py`](./mage_ai/orchestration/pipeline_scheduler_original.py) | `schedule_all()`、PipelineScheduler 类 |
| API 触发入口（管道） | [`mage_ai/server/api/triggers.py`](./mage_ai/server/api/triggers.py) | 完整管道 API 触发 |
| API 触发入口（Block） | [`mage_ai/server/api/runs.py`](./mage_ai/server/api/runs.py) | 单个 Block API 运行 |
| Trigger 工具 | [`mage_ai/orchestration/triggers/utils.py`](./mage_ai/orchestration/triggers/utils.py) | `create_and_start_pipeline_run` |
| 时间触发器 | [`mage_ai/orchestration/triggers/loop_time_trigger.py`](./mage_ai/orchestration/triggers/loop_time_trigger.py) | 定时轮询入口 |
| 事件触发器 | [`mage_ai/orchestration/triggers/event_trigger.py`](./mage_ai/orchestration/triggers/event_trigger.py) | 事件触发入口 |
| 并发配置 | [`mage_ai/orchestration/concurrency.py`](./mage_ai/orchestration/concurrency.py) | ConcurrencyConfig |
| 分布式锁 | [`mage_ai/orchestration/utils/distributed_lock.py`](./mage_ai/orchestration/utils/distributed_lock.py) | 分布式锁实现 |
| 状态检查 | [`mage_ai/orchestration/run_status_checker.py`](./mage_ai/orchestration/run_status_checker.py) | 运行状态查询工具 |

### 8.2 核心函数/方法

| 函数/方法 | 文件 | 行号 | 说明 |
|-----------|------|------|------|
| `schedule_all()` | pipeline_scheduler_original.py | L1656 | 全局调度入口，时间触发主循环 |
| `schedule_with_event()` | pipeline_scheduler_original.py | L2067 | 事件触发调度 |
| `sync_schedules()` | pipeline_scheduler_original.py | L2110 | 配置同步（yaml → DB） |
| `check_sla()` | pipeline_scheduler_original.py | L1606 | SLA 检查 |
| `configure_pipeline_run_payload()` | pipeline_scheduler_original.py | L1365 | 标准化 payload |
| `PipelineScheduler.start()` | pipeline_scheduler_original.py | L131 | 启动管道运行 |
| `PipelineScheduler.schedule()` | pipeline_scheduler_original.py | L213 | 调度 Block 执行 |
| `create_and_start_pipeline_run()` | triggers/utils.py | L83 | 创建并条件性启动运行 |
| `should_schedule()` | schedules.py | L537 | 单个调度是否需要执行 |
| `PipelineRun.create()` | schedules.py | L1353 | 创建运行（含去重逻辑） |
| `ApiTriggerPipelineHandler.post()` | server/api/triggers.py | L18 | API 触发入口 |

---

## 九、设计特点与关键理解点

### 设计优点

1. **统一调度入口**：所有触发方式（时间/事件/API）最终都通过 `schedule_all()` 的并发控制，行为一致
2. **异步解耦**：API 触发只创建 `INITIAL` 记录，不立即执行，避免阻塞 API 请求
3. **双层配置**：yaml 配置 + DB 持久化，兼顾代码化管理与运行时状态
4. **细粒度并发**：三级并发限制，灵活控制不同层面的并行度
5. **分布式锁**：原生支持多实例部署，避免重复调度

### 复杂度来源

1. **状态机组合**：PipelineRun 5 种状态 × BlockRun 8 种状态
2. **边界条件交织**：`start_time`、`last_enabled_at`、`landing_time`、skip 策略等多个条件共同作用
3. **两层触发体系**：Trigger（配置层）与 PipelineSchedule（运行层）的同步与映射
4. **并发策略叠加**：三级限制 × 两种超限策略

### 关键理解点

1. **Trigger ≠ PipelineSchedule**：Trigger 是"配置"，PipelineSchedule 是"运行时对象"，二者通过 `sync_schedules()` 同步
2. **API 触发不立即启动**：`should_schedule=False` 是关键设计，运行交给调度器统一管理
3. **去重 ≠ 并发控制**：`prevent_duplicates` 解决同一周期重复创建，并发限制控制同时运行数
4. **INITIAL 状态是排队池**：所有触发方式创建的运行都先进入 `INITIAL`，由 `schedule_all()` 按优先级和配额启动
5. **Landing Time 反转语义**：启用后 `start_time` 从"开始时间"变为"完成截止时间"
6. **两级超限策略独立**：单触发器限制和管道全触发器限制分别计算，超限行为取决于 `on_pipeline_run_limit_reached`

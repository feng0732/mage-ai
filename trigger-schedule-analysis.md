# Trigger 与 Schedule 代码实现分析

## 一、核心概念与架构分层

Trigger（触发器）与 Schedule（调度）是 Mage AI 中管道执行的两套核心机制，通过三层架构协作：

| 层级 | 概念 | 存储位置 | 核心职责 |
|------|------|----------|----------|
| 配置层 | **Trigger** | `triggers.yaml` | 定义"什么时候以什么方式触发管道执行" |
| 调度层 | **PipelineSchedule** | 数据库 `pipeline_schedule` 表 | Trigger 的运行时持久化表示，负责调度决策 |
| 执行层 | **PipelineRun / BlockRun** | 数据库 `pipeline_run` / `block_run` 表 | 每次触发产生的具体运行实例 |

**核心关联**：`sync_schedules()` 函数负责将代码中的 Trigger 配置同步到数据库的 PipelineSchedule 中。

---

## 二、API 触发完整流程分析

### 2.1 API 触发入口

API 触发有两个独立的入口点：

#### 入口一：完整管道触发
**文件**：`mage_ai/server/api/triggers.py`

**端点**：`POST /api/pipeline_schedules/{pipeline_schedule_id}/triggers`

**核心处理流程** (`ApiTriggerPipelineHandler.post()`):

```
请求到达
   ↓
鉴权检查 (token 验证)
   ├─ 检查 schedule_type == API
   ├─ 检查 token 非空
   └─ 验证 token 匹配
   ↓
构建 payload
   ├─ variables: URL 查询参数
   └─ event_variables: 请求体内容
   ↓
调用 create_and_start_pipeline_run(..., should_schedule=False)
   ↓
返回 PipelineRun
```

**关键代码**（`triggers.py` 第 18-65 行）：
- 第 35-40 行：Token 验证逻辑
- 第 46-53 行：event_variables 从请求体解析
- 第 58-63 行：调用 `create_and_start_pipeline_run`，**关键参数 `should_schedule=False`**

#### 入口二：单个 Block 运行触发
**文件**：`mage_ai/server/api/runs.py`

**端点**：`POST /api/runs`

这个入口用于直接运行单个 Block，不经过标准调度流程，直接执行并返回结果。

---

### 2.2 创建运行：`create_and_start_pipeline_run()`

**文件**：`mage_ai/orchestration/triggers/utils.py` 第 83-136 行

**函数签名**：
```python
def create_and_start_pipeline_run(
    pipeline: Pipeline,
    pipeline_schedule: PipelineSchedule,
    payload: Dict = None,
    should_schedule: bool = False,  # ← API 触发传入 False
    remote_blocks: List = None,
) -> PipelineRun:
```

**执行步骤**：

1. **配置 payload**（第 93-97 行）：
   - 调用 `configure_pipeline_run_payload()` 标准化 payload
   - 设置 `pipeline_schedule_id`、`pipeline_uuid`、`execution_date`
   - 注入 `execution_partition` 到 variables 中

2. **创建 PipelineRun**（第 106 行）：
   ```python
   pipeline_run = PipelineRun.create(**configured_payload)
   ```
   - **关键点**：没有使用 `prevent_duplicates` 参数，每次调用都会创建新运行
   - 创建后状态为 `INITIAL`

3. **条件性启动调度器**（第 108-131 行）：
   ```python
   if should_schedule:  # API 触发时为 False，跳过此分支
       pipeline_scheduler = PipelineScheduler(pipeline_run)
       pipeline_scheduler.start(should_schedule=should_schedule)
   ```

4. **更新 Schedule 状态**（第 133-134 行）：
   - 如果 PipelineSchedule 不是 ACTIVE 状态，自动激活

**设计意图**：
> API 触发故意不立即启动调度器，而是将 PipelineRun 留在 INITIAL 状态，
> 等待定时调度器 `schedule_all()` 统一处理。这样做的目的是：
> 1. 确保所有触发方式（时间/事件/API）都经过相同的并发控制逻辑
> 2. 避免 API 请求线程直接执行重量级调度操作

---

### 2.3 启动调度：`schedule_all()` 主循环

**文件**：`mage_ai/orchestration/pipeline_scheduler_original.py` 第 1656-1955 行

`schedule_all()` 是整个调度系统的心脏，由 `LoopTimeTrigger` 定时循环调用。

**处理 API 触发创建的 INITIAL 运行的流程**：

```
schedule_all() 被定时调用
   ↓
sync_schedules() 同步 yaml 配置到 DB
   ↓
获取所有 ACTIVE 状态的 PipelineSchedule
   ↓
按 pipeline_uuid 分组处理
   ↓
对每个 PipelineSchedule：
   ├─ 获取分布式锁 pipeline_schedule_{id}
   ├─ 调用 should_schedule() 判断是否需要新运行
   ├─ 获取 initial_pipeline_runs（INITIAL 状态的运行）
   │  ↑
   │  这就是 API 触发创建的运行！
   │
   ├─ 计算并发配额
   │  ├─ trigger_pipeline_run_limit（单触发器限制）
   │  ├─ running_pipeline_run_count（当前运行数）
   │  └─ pipeline_run_quota = limit - running_count
   │
   ├─ 按 execution_date 排序 initial_pipeline_runs
   ├─ 按配额选择要启动的运行
   └─ 超限的运行加入 excluded 列表
   ↓
管道级并发限制（全触发器）
   ├─ pipeline_run_limit_all_triggers
   └─ 再次过滤要启动的运行
   ↓
启动选中的运行：PipelineScheduler(r).start()
   ↓
处理超限运行：
   ├─ WAIT 策略：保留 INITIAL 状态，排队等待
   └─ SKIP 策略：设置为 CANCELLED
   ↓
处理所有 RUNNING 状态的运行：PipelineScheduler(r).schedule()
```

**关键代码位置**：
- 第 1796 行：`initial_pipeline_runs = pipeline_schedule.initial_pipeline_runs`
- 第 1872-1885 行：单触发器并发限制逻辑
- 第 1889-1906 行：管道全触发器并发限制逻辑
- 第 1908-1915 行：实际启动 PipelineRun
- 第 1919-1926 行：超限策略处理

---

### 2.4 PipelineScheduler 启动与调度

**文件**：`mage_ai/orchestration/pipeline_scheduler_original.py` 第 78-203 行

#### `start(should_schedule=True)` 方法（第 131-203 行）

```
获取分布式锁 pipeline_run_{id}
   ↓
检查 block_runs 是否已创建
   └─ 未创建则调用 create_block_runs()
   ↓
更新 PipelineRun 状态：INITIAL → RUNNING
   ↓
如果 should_schedule=True，调用 self.schedule()
   ↓
释放锁
```

**`should_schedule` 参数的含义**：
- `True`：启动后立即调用 `schedule()` 推进 Block 执行（时间触发使用）
- `False`：仅更新状态为 RUNNING，不立即调度 Block（仅特殊场景使用）

#### `schedule()` 方法（第 213 行起）

负责推进 RUNNING 状态 PipelineRun 的 Block 执行：
1. 检查超时（管道级、Block 级）
2. 检查 Block 失败级联
3. 调度可执行的 Block（`executable_block_runs()`）
4. 更新 BlockRun 状态

---

## 三、并发去重机制深度分析

### 3.1 去重机制：`prevent_duplicates`

**文件**：`mage_ai/orchestration/db/models/schedules.py` 第 1353-1388 行

**PipelineRun.create() 方法签名**：
```python
def create(
    self,
    create_block_runs: bool = True,
    prevent_duplicates: bool = False,  # ← 去重开关
    **kwargs,
) -> 'PipelineRun':
```

**去重逻辑**（第 1374-1381 行）：
```python
if prevent_duplicates:
    existing_pipeline_run = PipelineRun.query.filter(
        PipelineRun.execution_date == kwargs.get('execution_date'),
        PipelineRun.pipeline_schedule_id == kwargs.get('pipeline_schedule_id'),
        PipelineRun.pipeline_uuid == kwargs.get('pipeline_uuid'),
    ).first()
    if existing_pipeline_run is not None:
        return None  # 已存在则返回 None，不创建
```

**去重键**：`(execution_date, pipeline_schedule_id, pipeline_uuid)` 三元组

**使用场景对比**：

| 触发方式 | prevent_duplicates | 说明 |
|----------|-------------------|------|
| 时间触发 | **True** | 避免同一调度周期重复创建 |
| 事件触发 | **False** | 每次事件都是独立触发 |
| API 触发 | **False** | 每次 API 调用都是独立请求 |

> **为什么 API 触发不使用去重？**
> 
> API 触发的 `execution_date` 是 `datetime.utcnow()`，每次调用都不同，
> 三元组自然唯一，去重没有实际意义。用户期望每次 API 调用都产生一次运行。

---

### 3.2 并发控制：三级限制体系

**文件**：`mage_ai/orchestration/concurrency.py`

**ConcurrencyConfig 配置**：
```python
@dataclass
class ConcurrencyConfig(BaseConfig):
    block_run_limit: int = ...                    # Block 级
    pipeline_run_limit: int = ...                 # 单触发器级
    pipeline_run_limit_all_triggers: int = None   # 管道全触发器级
    on_pipeline_run_limit_reached: str = OnLimitReached.WAIT
```

#### 第一级：单触发器并发限制

**代码位置**：`pipeline_scheduler_original.py` 第 1759-1785 行

```python
trigger_pipeline_run_limit = pipeline_schedule.get_settings().pipeline_run_limit
if trigger_pipeline_run_limit is None:
    trigger_pipeline_run_limit = concurrency_config.pipeline_run_limit

# 计算配额
running_pipeline_run_count = pipeline_schedule.running_pipeline_run_count
pipeline_run_quota = trigger_pipeline_run_limit - running_pipeline_run_count

# 按配额筛选
if pipeline_run_quota > 0:
    initial_pipeline_runs.sort(key=lambda x: x.execution_date)
    pipeline_runs_to_start.extend(initial_pipeline_runs[:pipeline_run_quota])
    pipeline_runs_excluded_by_limit.extend(initial_pipeline_runs[pipeline_run_quota:])
```

**优先级规则**：按 `execution_date` 升序，先创建的先启动。

#### 第二级：管道全触发器并发限制

**代码位置**：`pipeline_scheduler_original.py` 第 1889-1906 行

```python
pipeline_run_limit = concurrency_config.pipeline_run_limit_all_triggers
if pipeline_run_limit is not None:
    pipeline_quota = pipeline_run_limit - len(
        pipeline_runs_by_pipeline.get(pipeline_uuid, [])
    )
    if pipeline_quota is not None:
        pipeline_quota = pipeline_quota if pipeline_quota > 0 else 0
        quota_filtered_runs = pipeline_runs_to_start[:pipeline_quota]
        pipeline_runs_excluded_by_limit.extend(pipeline_runs_to_start[pipeline_quota:])
```

#### 第三级：Block 级并发限制

在 `PipelineScheduler.schedule()` 中控制单管道内并发 Block 数。

---

### 3.3 超限策略：WAIT vs SKIP

**代码位置**：`pipeline_scheduler_original.py` 第 1917-1926 行

```python
if concurrency_config.on_pipeline_run_limit_reached == OnLimitReached.SKIP:
    for r in pipeline_runs_excluded_by_limit:
        pipeline_scheduler = PipelineScheduler(r)
        pipeline_scheduler.logger.warning(
            'Pipeline run limit reached... skipping this run',
            **pipeline_scheduler.build_tags(),
        )
        r.update(status=PipelineRun.PipelineRunStatus.CANCELLED)
```

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| **WAIT**（默认） | 保留 INITIAL 状态，下次 `schedule_all()` 时重试 | 周期性任务，延迟执行可接受 |
| **SKIP** | 设置为 CANCELLED，记录日志后丢弃 | 实时性要求高，错过时机的运行无意义 |

---

### 3.4 分布式锁机制

**文件**：`mage_ai/orchestration/utils/distributed_lock.py`

调度过程中使用的锁：

| 锁 Key | 保护范围 | 位置 |
|--------|----------|------|
| `pipeline_schedule_{id}` | 单个调度的决策过程 | 第 1755-1757 行 |
| `pipeline_run_{id}` | 单个运行的启动/调度过程 | 第 148-150 行 |

**作用**：防止多实例部署时重复调度、重复启动。

---

## 四、API 触发 → 创建运行 → 启动调度 → 并发去重 完整关联图

```
API 请求
   │
   ▼
┌─────────────────────────────────────────┐
│ ApiTriggerPipelineHandler.post()        │
│ mage_ai/server/api/triggers.py         │
│  ├─ Token 验证                          │
│  ├─ 构建 payload (variables, event)     │
│  └─ 调用 create_and_start_pipeline_run()│
│     should_schedule=False              │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ create_and_start_pipeline_run()         │
│ mage_ai/orchestration/triggers/utils.py │
│  ├─ configure_pipeline_run_payload()    │
│  ├─ PipelineRun.create()                │
│  │  └─ prevent_duplicates=False         │
│  │     → 状态: INITIAL                  │
│  ├─ should_schedule=False → 不启动      │
│  └─ 激活 PipelineSchedule (如未激活)    │
└─────────────────────┬───────────────────┘
                      │
                      │ 时间流逝...
                      │
                      ▼
┌─────────────────────────────────────────┐
│ schedule_all() [定时轮询]               │
│ mage_ai/orchestration/pipeline_scheduler_original.py │
│                                         │
│ 对每个 PipelineSchedule:                │
│  ├─ 获取锁 pipeline_schedule_{id}       │
│  ├─ should_schedule() 时间触发判断      │
│  ├─ initial_pipeline_runs ← API 触发的  │
│  ├─ 计算并发配额:                       │
│  │  ├─ trigger_pipeline_run_limit       │
│  │  ├─ pipeline_run_limit_all_triggers  │
│  │  └─ running_pipeline_run_count       │
│  ├─ 按配额筛选 + 排序                   │
│  └─ 超限处理: WAIT / SKIP               │
│                                         │
│ 启动选中的运行:                         │
│  └─ PipelineScheduler(r).start()        │
│     ├─ 创建 BlockRuns                   │
│     └─ 状态: INITIAL → RUNNING          │
│                                         │
│ 推进 RUNNING 状态的运行:                │
│  └─ PipelineScheduler(r).schedule()     │
│     ├─ 检查超时                         │
│     ├─ 调度可执行 Blocks                │
│     └─ 更新状态                         │
└─────────────────────────────────────────┘
```

---

## 五、核心对象模型

### 5.1 Trigger（触发器配置）

**文件**：`mage_ai/data_preparation/models/triggers/__init__.py`

基于 dataclass 的配置对象，存储在 `triggers.yaml` 中。

**核心字段**：
- `name`：触发器名称（与 pipeline_uuid 联合唯一）
- `schedule_type`：`TIME` / `EVENT` / `API`
- `schedule_interval`：`@once` / `@hourly` / `@daily` / cron 表达式
- `start_time`：开始时间
- `status`：`ACTIVE` / `INACTIVE`
- `last_enabled_at`：上次启用时间（关键边界字段）
- `variables` / `settings`：运行变量与高级设置

**SettingsConfig 高级设置**：
- `skip_if_previous_running`：前次运行未完成时跳过
- `allow_blocks_to_fail`：允许部分 Block 失败
- `create_initial_pipeline_run`：启用时立即创建初始运行
- `landing_time_enabled`：启用落地时间机制
- `pipeline_run_limit`：单触发器并发运行数限制
- `timeout`：超时时间（秒）

### 5.2 PipelineSchedule（调度任务）

**文件**：`mage_ai/orchestration/db/models/schedules.py` 第 86-766 行

数据库模型，是 Trigger 的运行时持久化形态。

**核心方法**：

| 方法 | 位置 | 作用 |
|------|------|------|
| `should_schedule()` | 第 537 行 | 判断是否应该调度新运行（核心决策） |
| `current_execution_date()` | 第 438 行 | 计算当前执行日期 |
| `next_execution_date()` | 第 510 行 | 计算下次执行日期 |
| `landing_time_enabled()` | 第 681 行 | 是否启用落地时间 |
| `active_schedules()` | 第 293 行 | 获取所有活跃调度 |

**`should_schedule()` 判断分支**：
```
状态检查 (ACTIVE?) → 开始时间检查 → 类型分支
  ├─ @once: 运行数为0 → True
  ├─ @always_on: 无运行或非活跃 → True
  └─ 其他时间间隔:
       ├─ 计算 current_execution_date
       ├─ last_enabled_at 边界检查
       ├─ start_time 检查
       ├─ 同 execution_date 去重检查
       └─ landing_time 预测判断
```

### 5.3 PipelineRun（管道运行实例）

**文件**：`mage_ai/orchestration/db/models/schedules.py` 第 768-1388 行

**状态枚举**：
- `INITIAL`：已创建未开始（API 触发后的初始状态）
- `RUNNING`：运行中
- `COMPLETED`：已完成
- `FAILED`：失败
- `CANCELLED`：已取消

**核心方法**：
- `create(prevent_duplicates=False)`：创建运行，支持去重
- `executable_block_runs()`：当前可执行的 Block 列表
- `update_block_run_statuses()`：级联更新 Block 状态

### 5.4 BlockRun（代码块运行实例）

**文件**：`mage_ai/orchestration/db/models/schedules.py` 第 1663-1738 行

**状态枚举**（8 种）：
`INITIAL` → `QUEUED` → `RUNNING` → `COMPLETED` / `FAILED` / `CANCELLED` / `UPSTREAM_FAILED` / `CONDITION_FAILED`

### 5.5 EventMatcher（事件匹配器）

**文件**：`mage_ai/orchestration/db/models/schedules.py` 第 1924-2023 行

用于事件触发模式，通过 JSON 模式递归匹配事件内容。

---

## 六、边界条件与特殊机制

### 6.1 `skip_if_previous_running`

**代码位置**：`pipeline_scheduler_original.py` 第 1835-1852 行

如果前一次运行还在进行中，不创建新运行，而是创建一个 CANCELLED 的占位记录。

```python
if pipeline_schedule.get_settings().skip_if_previous_running and (
    initial_pipeline_runs or running_pipeline_run_count > 0
):
    create_and_cancel_pipeline_run(
        pipeline,
        pipeline_schedule,
        payload,
        message='Previous pipeline run is still running... skipping this run',
    )
```

### 6.2 `last_enabled_at` 边界

**代码位置**：`schedules.py` 第 614-640 行

避免调度器启用时追溯历史执行日期：

```python
if last_enabled_at is not None:
    avoid_initial_pipeline_run = (
        compare(current_execution_date, last_enabled_at) == -1
        if not create_initial_pipeline_run
        else False
    )
```

### 6.3 Landing Time 机制

**代码位置**：`schedules.py` 第 660-676 行

确保管道在指定时间点前完成：
1. 收集最近 7 次历史运行时长
2. 计算 `平均运行时间 + 标准差/2` 作为缓冲
3. `当前时间 ≥ 预期执行时间 - (平均 + 缓冲)` → 开始调度

**注意**：启用 Landing Time 后，`start_time` 的语义从"开始时间"变为"完成截止时间"。

### 6.4 SLA 检查

**入口函数**：`check_sla()`，`pipeline_scheduler_original.py` 第 1606-1654 行

遍历进行中的 PipelineRun，比较 `execution_date + sla` 与当前时间，超时则标记 `passed_sla=True` 并发送通知。

### 6.5 两级超时

| 级别 | 配置位置 | 检查时机 |
|------|----------|----------|
| 管道超时 | `PipelineSchedule.settings.timeout` | 每次 `schedule()` 调用 |
| Block 超时 | `Block.timeout` | Block 调度循环中 |

超时后状态默认为 `FAILED`，可通过 `timeout_status` 自定义。

---

## 七、关键代码路径索引

### 7.1 核心文件

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| Trigger 配置 | `mage_ai/data_preparation/models/triggers/__init__.py` | Trigger 数据类、yaml 文件读写 |
| 调度模型 | `mage_ai/orchestration/db/models/schedules.py` | PipelineSchedule/PipelineRun/BlockRun |
| 主调度器 | `mage_ai/orchestration/pipeline_scheduler_original.py` | schedule_all()、PipelineScheduler 类 |
| API 触发入口 1 | `mage_ai/server/api/triggers.py` | 完整管道 API 触发 |
| API 触发入口 2 | `mage_ai/server/api/runs.py` | 单个 Block API 运行 |
| Trigger 工具 | `mage_ai/orchestration/triggers/utils.py` | create_and_start_pipeline_run |
| 时间触发器 | `mage_ai/orchestration/triggers/loop_time_trigger.py` | 定时轮询入口 |
| 事件触发器 | `mage_ai/orchestration/triggers/event_trigger.py` | 事件触发入口 |
| 并发配置 | `mage_ai/orchestration/concurrency.py` | ConcurrencyConfig |
| 状态检查 | `mage_ai/orchestration/run_status_checker.py` | 运行状态查询工具 |

### 7.2 核心函数

| 函数 | 文件 | 行号 | 说明 |
|------|------|------|------|
| `schedule_all()` | pipeline_scheduler_original.py | 1656 | 全局调度入口，时间触发主循环 |
| `schedule_with_event()` | pipeline_scheduler_original.py | 2067 | 事件触发调度 |
| `sync_schedules()` | pipeline_scheduler_original.py | 2110 | 配置同步（yaml → DB） |
| `check_sla()` | pipeline_scheduler_original.py | 1606 | SLA 检查 |
| `create_and_start_pipeline_run()` | triggers/utils.py | 83 | 创建并条件性启动运行 |
| `configure_pipeline_run_payload()` | pipeline_scheduler_original.py | 1365 | 标准化 payload |
| `should_schedule()` | schedules.py | 537 | 单个调度是否需要执行 |
| `PipelineRun.create()` | schedules.py | 1353 | 创建运行（含去重逻辑） |
| `PipelineScheduler.start()` | pipeline_scheduler_original.py | 131 | 启动管道运行 |
| `PipelineScheduler.schedule()` | pipeline_scheduler_original.py | 213 | 调度 Block 执行 |
| `ApiTriggerPipelineHandler.post()` | server/api/triggers.py | 18 | API 触发入口 |

---

## 八、设计特点与关键理解点

### 8.1 设计优点

1. **统一调度入口**：所有触发方式（时间/事件/API）最终都通过 `schedule_all()` 的并发控制逻辑，确保行为一致
2. **异步解耦**：API 触发只创建 INITIAL 记录，不立即执行，避免阻塞 API 请求
3. **双层配置**：yaml 配置 + DB 持久化，兼顾代码化管理与运行时状态
4. **细粒度并发**：三级并发限制，灵活控制不同层面的并行度
5. **分布式锁**：原生支持多实例部署，避免重复调度

### 8.2 复杂度来源

1. **状态机组合**：PipelineRun 5 种状态 × BlockRun 8 种状态，共 40 种组合
2. **边界条件交织**：start_time、last_enabled_at、landing_time、skip 策略等多个条件共同作用
3. **两层触发体系**：Trigger（配置层）与 PipelineSchedule（运行层）的同步与映射
4. **并发策略叠加**：三级限制 × 两种超限策略，行为推理成本高

### 8.3 关键理解点

1. **Trigger ≠ PipelineSchedule**：Trigger 是"配置"，PipelineSchedule 是"运行时对象"，二者通过 `sync_schedules()` 同步
2. **API 触发不立即启动**：`should_schedule=False` 是关键设计，将运行交给调度器统一管理
3. **去重 ≠ 并发控制**：`prevent_duplicates` 解决同一周期重复创建，并发限制控制同时运行数
4. **INITIAL 状态是排队池**：所有触发方式创建的运行都先进入 INITIAL，由 `schedule_all()` 按优先级和配额启动
5. **Landing Time 反转语义**：启用后 `start_time` 从"开始时间"变为"完成截止时间"
6. **两级超限策略独立**：单触发器限制和管道全触发器限制分别计算，超限行为取决于 `on_pipeline_run_limit_reached`

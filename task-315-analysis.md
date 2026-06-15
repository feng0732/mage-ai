# Mage AI Trigger 与 Schedule 实现分析

## 一、概述

Trigger（触发器）与 Schedule（调度）是 Mage AI 中管道执行的两套核心机制，二者通过"配置层 → 调度层 → 执行层"的三层架构协作：

- **Trigger** 是**配置层**概念，定义在每个管道的 `triggers.yaml` 文件中，描述"什么时候以什么方式触发管道执行"
- **Schedule** 是**调度层**概念，对应数据库中的 `pipeline_schedule` 表，是 Trigger 在运行时的持久化表示
- **PipelineRun** 是**执行层**概念，每次触发产生一次运行实例

核心关联：`sync_schedules()` 函数负责将代码中的 Trigger 配置同步到数据库的 PipelineSchedule 中。

---

## 二、核心对象模型

### 2.1 Trigger（触发器配置）

**文件位置**：[mage_ai/data_preparation/models/triggers/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/data_preparation/models/triggers/__init__.py)

Trigger 是基于 dataclass 的配置对象，存储在每个管道目录下的 `triggers.yaml` 文件中。

**核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | str | 触发器名称，与 pipeline_uuid 联合唯一 |
| `pipeline_uuid` | str | 关联的管道 UUID |
| `schedule_type` | ScheduleType | 类型：TIME / EVENT / API |
| `schedule_interval` | str | 调度间隔：@once / @hourly / @daily / @weekly / @monthly / @always_on 或 cron 表达式 |
| `start_time` | datetime | 开始时间 |
| `status` | ScheduleStatus | 状态：ACTIVE / INACTIVE |
| `last_enabled_at` | datetime | 上次启用时间 |
| `variables` | Dict | 运行时变量 |
| `settings` | Dict | 高级设置（并发控制、landing time 等） |
| `sla` | int | SLA 时间（秒） |
| `envs` | List | 环境过滤 |
| `token` | str | API 触发的安全令牌 |

**ScheduleType 枚举**：
- `TIME`：时间调度（最常用）
- `EVENT`：事件触发
- `API`：API 调用触发

**ScheduleInterval 枚举**：
- `@once`：只执行一次
- `@hourly`：每小时
- `@daily`：每天
- `@weekly`：每周
- `@monthly`：每月
- `@always_on`：常驻运行（用于流式管道）

**SettingsConfig 高级设置**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `skip_if_previous_running` | bool | False | 前一次运行未完成时跳过 |
| `allow_blocks_to_fail` | bool | False | 允许部分 Block 失败 |
| `create_initial_pipeline_run` | bool | False | 启用时立即创建初始运行 |
| `landing_time_enabled` | bool | False | 启用落地时间机制 |
| `pipeline_run_limit` | int | None | 单个触发器并发运行数限制 |
| `timeout` | int | None | 管道运行超时时间（秒） |
| `timeout_status` | str | None | 超时后的状态 |

### 2.2 PipelineSchedule（调度任务）

**文件位置**：[mage_ai/orchestration/db/models/schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/db/models/schedules.py#L86-L766)

PipelineSchedule 是数据库模型，对应 `pipeline_schedule` 表，是 Trigger 的运行时持久化形态。

**核心属性与方法**：

- `should_schedule()`：判断是否应该调度新的运行（核心方法）
- `current_execution_date()`：计算当前执行日期
- `next_execution_date()`：计算下次执行日期
- `landing_time_enabled()`：是否启用落地时间
- `pipeline_in_progress_runs_count`：进行中的运行数
- `runtime_history()`：历史运行时长（用于 landing time 预测）

**should_schedule() 判断逻辑**：

```
状态检查 → 开始时间检查 → 类型分支判断
  ├─ ONCE: 运行数为0 → True；流式且executor_count>1 → 可多实例
  ├─ ALWAYS_ON: 无运行 → True；非运行/非初始 → True
  └─ 其他时间间隔:
       ├─ 计算 current_execution_date
       ├─ last_enabled_at 检查（避免启用前的历史运行）
       ├─ start_time 检查
       ├─ 同 execution_date 的运行是否已存在
       └─ landing_time 额外判断（基于历史运行时长预测）
```

### 2.3 PipelineRun（管道运行实例）

**文件位置**：[mage_ai/orchestration/db/models/schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/db/models/schedules.py#L768-L1326)

PipelineRun 是管道的一次具体执行，对应 `pipeline_run` 表。

**状态枚举 PipelineRunStatus**：
- `INITIAL`：初始状态，已创建未开始
- `RUNNING`：运行中
- `COMPLETED`：已完成
- `FAILED`：失败
- `CANCELLED`：已取消

**核心属性**：
- `execution_date`：执行日期（调度的逻辑时间）
- `executable_block_runs()`：当前可执行的 Block 列表
- `update_block_run_statuses()`：更新 Block 运行状态（级联失败）

### 2.4 BlockRun（代码块运行实例）

**文件位置**：[mage_ai/orchestration/db/models/schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1663-L1738)

BlockRun 是管道中单个 Block 的运行实例，对应 `block_run` 表。

**状态枚举 BlockRunStatus**：
- `INITIAL`：初始
- `QUEUED`：已入队
- `RUNNING`：运行中
- `COMPLETED`：完成
- `FAILED`：失败
- `CANCELLED`：取消
- `UPSTREAM_FAILED`：上游失败（级联）
- `CONDITION_FAILED`：条件不满足

### 2.5 EventMatcher（事件匹配器）

**文件位置**：[mage_ai/orchestration/db/models/schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/db/models/schedules.py#L1924-L2023)

EventMatcher 用于事件触发模式，通过 JSON 模式匹配事件内容。

**核心方法**：
- `match(config)`：递归匹配事件字典与 pattern
- `active_pipeline_schedules()`：获取关联的活跃调度
- `active_event_matchers()`：获取所有活跃的事件匹配器

---

## 三、调度过程

### 3.1 整体架构图

```
                ┌─────────────────┐
                │  triggers.yaml  │  (配置层: Trigger)
                └────────┬────────┘
                         │ sync_schedules()
                         ▼
                ┌─────────────────┐
                │ PipelineSchedule│  (调度层: DB 模型)
                └────────┬────────┘
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    TimeTrigger    EventTrigger   API 触发
    (定时轮询)     (事件驱动)    (主动调用)
         │               │               │
         └───────────────┼───────────────┘
                         ▼
                ┌─────────────────┐
                │   PipelineRun   │  (执行层: 运行实例)
                └────────┬────────┘
                         ▼
                ┌─────────────────┐
                │    BlockRun     │  (细粒度执行)
                └─────────────────┘
```

### 3.2 时间触发流程

**入口**：[LoopTimeTrigger](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/triggers/loop_time_trigger.py)

**核心流程**（`schedule_all()` 函数）：

1. **同步配置**：调用 `sync_schedules()` 将 `triggers.yaml` 中的 Trigger 配置同步到数据库

2. **获取活跃调度**：`PipelineSchedule.active_schedules()` 获取所有 ACTIVE 状态的调度

3. **按管道分组处理**：
   - 按 pipeline_uuid 分组，便于处理管道级并发限制
   - 对每个调度获取分布式锁（`pipeline_schedule_{id}`）

4. **调度决策**：
   - 调用 `pipeline_schedule.should_schedule()` 判断是否需要新运行
   - 处理流式管道重启（requirements 变更时）

5. **创建 PipelineRun**：
   - 检查 `skip_if_previous_running` 配置
   - 创建 INITIAL 状态的 PipelineRun
   - 记录 git sync 状态（如果启用）

6. **并发限制**：
   - 单触发器级别：`pipeline_run_limit`
   - 管道全触发器级别：`pipeline_run_limit_all_triggers`
   - 超限策略：WAIT（排队）或 SKIP（取消）

7. **启动运行**：
   - 调用 `PipelineScheduler(pipeline_run).start()`
   - 创建 BlockRun 实例
   - 更新状态为 RUNNING

8. **调度运行中的管道**：
   - 遍历所有 RUNNING 状态的 PipelineRun
   - 调用 `PipelineScheduler(r).schedule()` 推进 Block 执行

### 3.3 事件触发流程

**入口**：[EventTrigger](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/triggers/event_trigger.py)

**核心流程**（`schedule_with_event()` 函数）：

1. 获取所有活跃的 EventMatcher
2. 遍历匹配器，调用 `e.match(event)` 进行模式匹配
3. 对匹配成功的调度，创建 PipelineRun
4. 事件数据通过 `variables.event` 传递给管道

### 3.4 API 触发流程

通过 API 端点（`/api/pipelines/{pipeline_uuid}/runs`）直接触发，创建对应 PipelineSchedule 的运行实例。

### 3.5 PipelineScheduler 调度器

**文件位置**：[mage_ai/orchestration/pipeline_scheduler_original.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py)

`PipelineScheduler` 类负责单个 PipelineRun 的生命周期管理：

**核心方法**：
- `start()`：启动管道运行，初始化 BlockRun
- `schedule()`：调度可执行的 Block，推进执行
- `on_block_complete()`：Block 完成回调，继续调度
- `on_block_failure()`：Block 失败回调，处理级联失败
- `stop()`：停止管道运行

**状态流转**：

```
PipelineRun:
  INITIAL → RUNNING → COMPLETED
                      → FAILED (Block 失败 / 超时)
                      → CANCELLED (手动取消)

BlockRun:
  INITIAL → QUEUED → RUNNING → COMPLETED
                                  → FAILED
                                  → UPSTREAM_FAILED (级联)
                                  → CONDITION_FAILED
```

---

## 四、边界判断逻辑

### 4.1 并发控制

**配置文件**：[concurrency.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/concurrency.py)

**三级并发限制**：

| 级别 | 配置项 | 说明 |
|------|--------|------|
| 单触发器 | `settings.pipeline_run_limit` | 单个 PipelineSchedule 的并发运行数 |
| 单管道（全触发器） | `pipeline_run_limit_all_triggers` | 一个管道所有触发器的总并发数 |
| Block 级别 | `block_run_limit` | 单管道内并发 Block 数 |

**超限策略**（`on_pipeline_run_limit_reached`）：
- `WAIT`：等待，保持 INITIAL 状态排队
- `SKIP`：跳过，直接设置为 CANCELLED

**分布式锁**：
使用 `DistributedLock` 防止多实例调度器重复调度，锁 key 包括：
- `pipeline_schedule_{id}`：调度级锁
- `pipeline_run_{id}`：运行级锁

### 4.2 时区与时间计算

**关键函数**：`current_execution_date()` 和 `next_execution_date()`

**执行日期计算规则**：

| 间隔类型 | 计算方式 |
|----------|----------|
| @once / @always_on | 当前时间 |
| @hourly | 整点（分秒清零） |
| @daily | 当天 0 点（时分秒清零） |
| @weekly | 本周一 0 点 |
| @monthly | 本月 1 号 0 点 |
| cron 表达式 | 通过 croniter 计算上一个匹配时间 |

**时区处理**：
- 数据库存储带时区信息（`DateTime(timezone=True)`）
- 比较时统一转换为 UTC（`pytz.UTC`）

### 4.3 Landing Time 机制

**作用**：确保管道在指定时间点前完成执行（落地时间）。

**判断逻辑**（`should_schedule()` 中）：

1. 收集最近 N 次历史运行时长（默认 7 次）
2. 计算平均运行时间 + 标准差/2 作为缓冲
3. 如果当前时间 ≥ 预期执行时间 - (平均运行时间 + 缓冲)，则开始调度

**特点**：
- 仅对 TIME 类型且间隔为 HOURLY/DAILY/WEEKLY/MONTHLY 生效
- `start_time` 被解释为"完成截止时间"而非"开始时间"

### 4.4 SLA 检查

**入口函数**：`check_sla()`

**逻辑**：
- 遍历所有进行中的 PipelineRun
- 比较 `execution_date + sla` 与当前时间
- 超时则设置 `passed_sla = True` 并发送通知

### 4.5 超时处理

**两级超时**：

| 级别 | 配置位置 | 检查时机 |
|------|----------|----------|
| 管道超时 | PipelineSchedule.settings.timeout | 每次 schedule() 调用 |
| Block 超时 | Block.timeout | Block 调度循环中 |

超时后的状态：
- 默认：FAILED
- 可通过 `timeout_status` 自定义

### 4.6 skip_if_previous_running

**作用**：如果前一次运行还在进行中，跳过本次调度。

**行为**：
- 创建一个 CANCELLED 状态的 PipelineRun 作为占位记录
- 日志记录 "Previous pipeline run is still running... skipping this run"

### 4.7 last_enabled_at 边界

**作用**：避免调度器启用时追溯历史执行日期。

**逻辑**：
- 如果 `last_enabled_at` 存在且 `create_initial_pipeline_run` 为 False
- 且 `current_execution_date < last_enabled_at`
- 则不创建运行，等待下一个执行周期

### 4.8 去重机制

PipelineRun 创建时使用 `prevent_duplicates` 参数，基于 `execution_date + pipeline_schedule_id` 联合去重，避免同一调度周期重复创建运行。

---

## 五、关键代码路径索引

### 5.1 核心文件

| 模块 | 文件路径 | 主要职责 |
|------|----------|----------|
| Trigger 配置 | [mage_ai/data_preparation/models/triggers/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/data_preparation/models/triggers/__init__.py) | Trigger 数据类、文件读写 |
| 调度模型 | [mage_ai/orchestration/db/models/schedules.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/db/models/schedules.py) | PipelineSchedule/PipelineRun/BlockRun/EventMapper |
| 主调度器 | [mage_ai/orchestration/pipeline_scheduler_original.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/pipeline_scheduler_original.py) | schedule_all()、PipelineScheduler 类 |
| 时间触发器 | [mage_ai/orchestration/triggers/loop_time_trigger.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/triggers/loop_time_trigger.py) | 定时轮询入口 |
| 事件触发器 | [mage_ai/orchestration/triggers/event_trigger.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/triggers/event_trigger.py) | 事件触发入口 |
| 并发配置 | [mage_ai/orchestration/concurrency.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/concurrency.py) | ConcurrencyConfig |
| 状态检查 | [mage_ai/orchestration/run_status_checker.py](file:///d:/fz/0601/solo-dogfeeding/code/315-mage-ai/mage_ai/orchestration/run_status_checker.py) | 运行状态查询工具 |

### 5.2 核心函数

| 函数 | 位置 | 说明 |
|------|------|------|
| `schedule_all()` | pipeline_scheduler_original.py:1656 | 全局调度入口，时间触发主循环 |
| `schedule_with_event()` | pipeline_scheduler_original.py:2067 | 事件触发调度 |
| `sync_schedules()` | pipeline_scheduler_original.py:2110 | 配置同步（yaml → DB） |
| `check_sla()` | pipeline_scheduler_original.py:1606 | SLA 检查 |
| `should_schedule()` | schedules.py:537 | 单个调度是否需要执行 |
| `PipelineScheduler.start()` | pipeline_scheduler_original.py:130 | 启动管道运行 |
| `PipelineScheduler.schedule()` | - | 调度 Block 执行 |
| `EventMatcher.match()` | schedules.py:2007 | 事件模式匹配 |

---

## 六、设计特点与权衡

### 优点
1. **双层配置**：yaml 配置 + DB 持久化，兼顾代码化管理与运行时状态
2. **多触发模式**：支持时间、事件、API 三种触发方式
3. **细粒度并发控制**：三级并发限制（触发器级、管道级、Block 级）
4. **Landing Time**：智能预测启动时间，确保按时完成
5. **分布式锁**：支持多实例部署，避免重复调度

### 潜在复杂度
1. **状态机较多**：PipelineRun 5 种状态 + BlockRun 8 种状态，组合复杂
2. **边界条件多**：start_time、last_enabled_at、landing_time、skip 策略等交织
3. **两层触发体系**：Trigger（配置）与 PipelineSchedule（运行时）的同步容易混淆
4. **并发策略组合**：WAIT/SKIP + 多级限制，行为推理成本较高

### 关键理解点
- Trigger 是"配置"，PipelineSchedule 是"运行时对象"，二者通过 `sync_schedules()` 同步
- `should_schedule()` 是调度决策的核心，集成了时间、状态、历史等多维度判断
- Landing Time 模式下 `start_time` 的语义会从"开始时间"变为"完成截止时间"
- 并发限制在触发器级和管道级分别生效，超限时的行为取决于 `on_pipeline_run_limit_reached`

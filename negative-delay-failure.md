# delay 为负值时的完整失败链路深度解析

## 一、前置事实：time.sleep(负数) 会抛 ValueError

### 1.1 实测验证

Python 版本: `Python 3.14.4`

```
>>> import time
>>> time.sleep(-5)
Traceback (most recent call last):
  File "<string>", line 1, in <module>
    import time; time.sleep(-5)
                 ~~~~~~~~~~^^^^
ValueError: sleep length must be non-negative
```

**结论**: `time.sleep(负数)` **不**会像之前误以为的那样变成 `sleep(0)`，而是直接抛出 `ValueError`。这是修正分析的最关键前提。

### 1.2 关键代码位置

@retry 装饰器核心逻辑: [shared/retry.py#L27-L60](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/shared/retry.py#L27-L60)

```python
def retry_func(*args, **kwargs):
    max_attempts = retries + 1
    attempt = 1
    total_delay = 0
    curr_delay = delay              # ← 如果 delay<0，curr_delay 初始就是负值
    while attempt <= max_attempts:
        try:
            # ① 执行被装饰的函数
            return func(*args, **kwargs)
        except Exception as e:      # ← 捕获 ① 的异常
            # ② 记录 retry warning 日志（含原始异常 e）
            logger.warning(...)
            attempt += 1
            # ③ 判断是否满足终止条件
            if attempt > max_attempts or total_delay + curr_delay >= max_delay:
                raise e             # ← 重新抛出原始异常
            # ④ 等待——如果 delay<0，这里会抛出 ValueError！
            time.sleep(curr_delay)
            total_delay += curr_delay
            if exponential_backoff:
                curr_delay *= 2
    return func(*args, **kwargs)
```

---

## 二、场景一：retries=3, delay=-5, max_delay=60, exponential_backoff=True

这是最典型的"配置了正常 retries，但手误把 delay 写成负数"的场景。

### 2.1 逐行精确推演

**初始状态**:
```python
max_attempts = 3 + 1 = 4
attempt = 1
total_delay = 0
curr_delay = -5
```

---

**第一轮迭代 (attempt=1)**:

```
进入 while 1 <= 4: True
    try:
        return func()              # ① func 抛出原始异常 e1 (如 KafkaError: Connection refused)
    except Exception as e:         # e = e1
        logger.warning(
            'Exception ... attempt 1 of 4',
            error=e1,              # 日志里记录的是原始异常 e1
            **tags
        )                          # ② 第一条 warning 日志 ✅
        attempt = 1 + 1 = 2

        # ③ 判断终止条件:
        attempt > max_attempts?   → 2 > 4? → False
        total_delay + curr_delay >= max_delay?
            → 0 + (-5) = -5 >= 60? → False
        → 两个条件都不满足，不终止

        # ④ 关键转折点：sleep 负值
        time.sleep(-5)             # 抛出 ValueError("sleep length must be non-negative")
        ↑↑↑ 这里已经不在 except 块内了！ValueError 没有被捕获，直接向外层冒泡
```

**💥 第一轮就结束了，后续代码全部不执行：**
```python
        time.sleep(curr_delay)  # ← 从这里抛出，后面三行完全不执行
        total_delay += curr_delay
        if exponential_backoff:
            curr_delay *= 2
```

### 2.2 异常覆盖关系

```
@retry 装饰器内
    │
    ├─ try:
    │    func() → 抛出 e1 (KafkaError)
    │
    └─ except Exception as e:  ← 捕获 e1
           │
           ├─ logger.warning(error=e1)  ← 日志记录了原始异常 ✅
           │
           └─ 离开 except 块
                │
                └─ time.sleep(-5) → 抛出 e2 (ValueError)
                                   ↑ 不在任何 try/except 内，直接冒泡
```

**最终向上层抛出的异常是 e2 (ValueError)，不是原始异常 e1！**

原始异常 e1 的信息**只存在于 warning 日志中**，不会出现在最终的异常栈追踪里。

### 2.3 Streaming Pipeline 外层接收到什么

回到 [streaming_pipeline_executor.py#L96-L136](file:///d:/fz/0601/solo-dogfeeding/code/313-mage-ai/mage_ai/data_preparation/executors/streaming_pipeline_executor.py#L96-L136)：

```python
try:
    ...
    __execute_with_retry()   # ← 这里抛出的是 ValueError("sleep length must be non-negative")
except Exception as e:       # e = ValueError
    if not build_block_output_stdout:
        self.logger.exception(
            f'Failed to execute streaming pipeline {self.pipeline.uuid}',
            error=ValueError("sleep length must be non-negative"),  # ← 🔴 记录的是 ValueError！
            **tags
        )
    if not infinite_retries:
        self.__update_pipeline_run_status(
            pipeline_run_id,
            PipelineRun.PipelineRunStatus.FAILED,
            error=ValueError(...),  # ← 🔴 存入数据库的错误也是 ValueError
        )
    raise e  # 向上抛出的是 ValueError
```

### 2.4 全景信息汇总

| 信息位置 | 记录的异常 | 内容正确吗？ |
|---------|----------|------------|
| @retry 内 warning 日志（第 1 条） | ✅ e1 (KafkaError) | 正确，是真实原因 |
| StreamingExecutor exception 日志 | ❌ e2 (ValueError) | 误导性，掩盖了真实原因 |
| PipelineRun.error 字段 | ❌ e2 (ValueError) | 误导性，用户看不到真实原因 |
| PipelineRun.stacktrace 字段 | ❌ e2 的栈追踪 | 误导性，栈追踪指向 time.sleep |
| 最终向上抛出的异常 | ❌ e2 (ValueError) | 误导性 |

---

## 三、场景二：delay=-5，但终止条件刚好在 sleep 前触发

当 `retries` 很小或 `max_delay` 也为负时，终止条件判断 `attempt > max_attempts` 可能先于 sleep 触发。

**配置**: `retries=0, delay=-5, max_delay=60`

```python
max_attempts = 0 + 1 = 1
attempt = 1
curr_delay = -5

进入 while 1 <= 1: True
    try:
        func() → 抛出 e1
    except Exception as e:
        logger.warning('attempt 1 of 1', error=e1)  # ✅ 日志有原始异常
        attempt = 1 + 1 = 2

        # 终止条件判断:
        attempt > max_attempts? → 2 > 1? → True!
        → 直接 raise e1，走不到 time.sleep
```

**结果**: 正常从第 54 行抛出原始异常 e1，ValueError 没有机会出场。

---

## 四、场景三：max_delay 也为负值的叠加情况

**配置**: `retries=3, delay=-5, max_delay=-60`

```python
max_attempts = 4
attempt = 1
curr_delay = -5
total_delay = 0

进入 while 1 <= 4: True
    try:
        func() → 抛出 e1
    except Exception as e:
        logger.warning('attempt 1 of 4', error=e1)
        attempt = 2

        # 终止条件判断:
        attempt > max_attempts? → 2 > 4? → False
        total_delay + curr_delay >= max_delay?
            → 0 + (-5) = -5 >= -60? → True! (负数比较，-5 比 -60 大)

        → raise e1，走不到 sleep
```

**结果**: 因为 `max_delay` 也是负数，`-5 >= -60` 为 True，终止条件触发，正常抛出 e1。不会遇到 ValueError。

---

## 五、场景四：exponential_backoff=false，delay 负值较小

**配置**: `retries=5, delay=-1, max_delay=60, exponential_backoff=false`

curr_delay 始终保持 -1，不会增长

```
Round 1 (attempt=1):
    func() → e1
    warning(attempt 1/5, error=e1)
    attempt=2
    2>6? No.  0+(-1)=-1 >=60? No.
    time.sleep(-1) → ValueError e2！💥
```

无论 delay 负得多小（-1, -0.1 等），只要是负数且没被终止条件拦住，第一次 sleep 就炸。

---

## 六、什么时候会遇到 ValueError？（触发条件总结）

进入 `time.sleep(负值)` 的**充要条件**是：在第 1 次失败后的终止条件判断中，以下两个同时为 False：

```
条件 A: attempt > max_attempts         → False（重试次数还没耗尽）
条件 B: total_delay + curr_delay >= max_delay  → False（累计延迟判断不成立）
```

| 配置场景 | 会不会进入 sleep？ | 抛出的异常 |
|---------|-----------------|----------|
| retries=3, delay=-5, max_delay=60 | ✅ 会 | ValueError（掩盖 e1） |
| retries=0, delay=-5, max_delay=60 | ❌ 不会（attempt 先耗尽） | 原始异常 e1 |
| retries=3, delay=-5, max_delay=-60 | ❌ 不会（-5 >= -60） | 原始异常 e1 |
| retries=1, delay=-5, max_delay=60 | ✅ 会（第1次后不终止） | ValueError |
| retries=1, delay=-5, max_delay=3 | ❌ 不会（0+(-5)=-5>=3? No，但是：attempt=2>2? No，再看 -5>=3 No → 会 sleep！等等，让我重算 |

让我精确计算最后一个：`retries=1, delay=-5, max_delay=3`

```
max_attempts = 2
Round 1 (attempt=1):
    func() 抛出 e1
    warning(attempt 1/2, error=e1)
    attempt = 2
    attempt > max_attempts? → 2 > 2? → False
    total_delay + curr_delay >= max_delay? → 0 + (-5) = -5 >= 3? → False
    → 两个都是 False，进入 sleep(-5) → ValueError 💥
```

**结论**: 只要 `retries >= 1` 且 `max_delay > 0`（正常正数），delay 为负值就一定**会**遇到 ValueError 掩盖原始异常。

---

## 七、状态与日志变化的时间线

以最典型的场景 `retries=3, delay=-5, max_delay=60, infinite_retries=False` 为例：

### 时间线

```
时间点 T0:
  __execute_with_retry() 开始执行
  retry_metadata['attempts'] = 1（由第 36 行更新）

时间点 T1 (func 失败):
  ↳ func() 抛出 KafkaError("Connection refused") = e1
  ↳ 进入 except 块

时间点 T2 (记录第 1 条 warning 日志):
  ↳ logger.warning('attempt 1 of 4', error=e1)
  ↳ 日志内容：真实原因 KafkaError ✅
  ↳ attempt 更新为 2

时间点 T3 (终止条件判断，不满足):
  ↳ 2 > 4? No
  ↳ -5 >= 60? No
  ↳ 继续往下执行

时间点 T4 (💥 发生 ValueError):
  ↳ time.sleep(-5) 抛出 ValueError("sleep length must be non-negative") = e2
  ↳ 此时已不在 except 块内，e2 直接冒泡
  ↳ ❌ total_delay 仍为 0（没来得及 +=）
  ↳ ❌ curr_delay 仍为 -5（没来得及 *= 2）
  ↳ ❌ 只执行了 1 次 func，而不是 retries+1=4 次
  ↳ ❌ retry_metadata['attempts'] 只被更新了 1 次

时间点 T5 (StreamingExecutor 外层 except 捕获 e2):
  ↳ self.logger.exception('Failed to execute ...', error=ValueError(...))
  ↳ 日志内容：ValueError ❌，看不到真实原因 KafkaError

时间点 T6 (更新 PipelineRun 状态):
  ↳ infinite_retries=False（因为配置了非空 retry_config）
  ↳ PipelineRun.status = FAILED
  ↳ PipelineRun.completed_at = now
  ↳ PipelineRun.error = "ValueError: sleep length must be non-negative" ❌
  ↳ PipelineRun.stacktrace = ValueError 的栈追踪（指向 time.sleep） ❌
  ↳ 发送失败通知，通知里的错误也是 ValueError ❌

时间点 T7:
  ↳ raise e2，整个 execute() 异常退出
```

---

## 八、关键风险点

### 8.1 异常信息被掩盖

用户排查问题时看到的是：
- 数据库里：`ValueError: sleep length must be non-negative`
- 最终异常栈：指向 `time.sleep`
- 看起来像是 Mage 框架的 bug

但真正原因（Kafka 连接失败 / Postgres 连不上 / 业务代码异常）**只在 warning 日志里出现了一次**，如果用户只看 ERROR 级别日志，会完全错过。

### 8.2 实际重试次数与配置不符

配置了 `retries=3`，实际只执行了 **1 次** func。用户以为会重试 4 次，但因为 ValueError 提前炸了，根本没进行重试。

### 8.3 total_delay / curr_delay 不完整

因为 ValueError 发生在 `time.sleep()` 这行，后面的 `total_delay += curr_delay` 和 `curr_delay *= 2` 都没执行。如果有外部代码依赖这些状态（目前没有，但未来可能有），会是脏数据。

### 8.4 retry_metadata 不准确

`retry_metadata['attempts']` 只被更新了 1 次（值为 1），不是预期的 4 次。

---

## 九、代码缺陷定位

**根本问题**：`time.sleep(curr_delay)` 位于 except 块之外，且没有自己的 try/except 保护。

```python
# shared/retry.py 第 52-58 行
attempt += 1
if attempt > max_attempts or total_delay + curr_delay >= max_delay:
    raise e
time.sleep(curr_delay)              # ← 🔴 这里裸奔，没有任何异常保护
total_delay += curr_delay           # ← 上面抛异常的话，这行及以下全不执行
if exponential_backoff:
    curr_delay *= 2
```

**建议修复方向**（仅分析，不改代码）：
1. 在进入重试循环前先做参数校验：`if retries < 0 or delay < 0 or max_delay < 0: raise ValueError(...)`
2. 或者给 `time.sleep` 包一层 try/except，捕获 ValueError 后要么转为合理提示，要么将原始异常链带出来
3. 或者用 `time.sleep(max(0, curr_delay))` 做防御性截断，但这会掩盖配置错误，不如直接报错

# Pipeline 编辑同步保存阶段调用参数详解

## 一、两条保存入口总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│           入口 A：异步编辑保存（前端编辑页面 → 全量保存）                   │
│                                                                          │
│  前端 savePipelineContent()                                              │
│  → PUT /api/pipelines/:uuid?update_content=true                         │
│  → PipelineResource.update()                                            │
│  → Pipeline.update(data, update_content=True)                           │
│      ├─ 调用① save_async()              ← 无参，全量 self.to_dict()     │
│      └─ 调用③ save_async(block_type=,   ← 无 block_uuid，               │
│                         widget=)           仍是全量 self.to_dict()       │
│                                                                          │
│  结果：metadata.yaml 被全量覆盖写入，block_uuid 分支不触发                │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│           入口 B：同步单区块保存（Block 属性/关系变更 → 精确保存）          │
│                                                                          │
│  Block.update() / Block.__update_pipeline_block()                       │
│  → Pipeline.update_block(block, widget=widget)                          │
│      ├─ 有 upstream/callback/conditional/downstream 参数                 │
│      │   → save_kwargs 为空 → save(block_type=, extension_uuid=,       │
│      │                            widget=)  ← 无 block_uuid，全量写盘    │
│      └─ else（只改 block 自身属性）                                      │
│          → save_kwargs = {block_uuid: block.uuid}                       │
│          → save(block_type=, block_uuid=, extension_uuid=, widget=)     │
│            ← 有 block_uuid，走"先读磁盘再合并"精确保存分支               │
│                                                                          │
│  Pipeline.add_block() / delete_block()                                   │
│  → save()  ← 无 block_uuid，全量写盘                                    │
│                                                                          │
│  结果：部分场景走精确保存（先读磁盘再合并），部分场景全量写盘              │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 二、入口 A：异步编辑保存 —— 参数逐层传递

### 2.1 前端：savePipelineContent → updatePipeline

`savePipelineContent`（`edit.tsx` L1187-L1446）构造 payload：

```typescript
const updatedPipeline = {
  ...pipeline,                          // 来自 GET /pipelines/:uuid 的最新快照
  ...pipelineOverride,                  // 调用方传入的覆盖（如 name, type, extensions）
  blocks: blocksToSave,                 // 普通 block 数组（含 content, outputs）
  callbacks: callbacksToSave,           // callback block 数组
  conditionals: conditionalsToSave,     // conditional block 数组
  extensions: extensionsToSave,         // { ext_uuid: { blocks: [...] } }
  widgets: [...],                       // chart block 数组
};
delete updatedPipeline.updated_at;

return updatePipeline({ pipeline: updatedPipeline });
```

`updatePipeline` mutation（`edit.tsx` L1148-L1149）：

```typescript
api.pipelines.useUpdate(pipelineUUID, { update_content: true })
// HTTP: PUT /api/pipelines/:pipelineUUID?update_content=true
// Body: { pipeline: updatedPipeline }
```

**关键参数**：
- URL query `update_content=true` —— 唯一控制后端"是否更新 block 文件"的开关
- **没有** `block_uuid` 等参数 —— 永远是全量保存

### 2.2 API 层：PipelineResource.update

`PipelineResource.update`（`PipelineResource.py` L653-L876）：

```python
# L670-L672: 解析 update_content
update_content = query.get('update_content', [False])
if update_content:
    update_content = update_content[0]   # True

# L740-L743: 转调模型层
await self.model.update(
    ignore_keys(payload, ['add_upstream_for_block_uuid']),
    update_content=update_content,       # 透传给 Pipeline.update()
)
```

传入 `Pipeline.update()` 的 `data` 参数是整个 `pipeline` 对象（外层 key 已由 BaseResource 剥掉），包含 `blocks`/`callbacks`/`conditionals`/`extensions`/`widgets` 等完整数据。

### 2.3 模型层：Pipeline.update(data, update_content=True)

`Pipeline.update`（`pipeline.py` L1246-L1589）内部有 **3 次** `save_async` 调用：

#### 调用 ①：L1277 —— pipeline 重命名时

```python
await self.save_async()
# 无参数！block_uuid=None → 走 else 分支 → 全量序列化 self.to_dict() 写盘
```

**场景**：仅当 `data['name']` 改变触发重命名时执行。

#### 调用 ②：L1351 —— 元数据变化时

```python
if should_save:
    await self.save_async()
# 无参数！block_uuid=None → 走 else 分支 → 全量序列化 self.to_dict() 写盘
```

**场景**：extensions / tags / description / type / executor_type / retry_config / settings / block 顺序等任一变化。

#### 调用 ③：L1532-L1536 —— block 内容变化时（update_content=True 分支）

```python
if should_save_async:
    await self.save_async(
        block_type=block.type,
        widget=widget,
    )
# ⚠️ 缺少 block_uuid 和 extension_uuid！
# block_uuid=None → 走 else 分支，block_type 和 widget 是死参数
```

**这是最频繁调用的 save_async**，每个有内容变化的 block 都会触发一次。

---

## 三、入口 B：同步单区块保存 —— 关键路径

### 3.1 触发路径：Block.update() → Pipeline.update_block()

当用户通过 API 或交互修改单个 block 的属性时，调用链为：

```
BlockResource.update(payload)
  → Block.update(data)
    → Block.__update_pipeline_block()    [改 color/configuration/has_callback 等]
      → Pipeline.update_block(block, widget=widget)
```

### 3.2 Pipeline.update_block 的 save_kwargs 机制

`Pipeline.update_block`（`pipeline.py` L2008-L2143）根据变更类型决定是否传入 `block_uuid`：

```python
save_kwargs = dict()                     # L2018: 初始为空

extension_uuid = block.extension_uuid    # L2020: 从 block 实例读取
is_callback = BlockType.CALLBACK == block.type
is_conditional = BlockType.CONDITIONAL == block.type
is_extension = BlockType.EXTENSION == block.type

if upstream_block_uuids is not None:     # 改上游关系
    ...                                  # save_kwargs 为空
elif callback_block_uuids is not None:   # 改回调
    ...                                  # save_kwargs 为空
elif conditional_block_uuids is not None:# 改条件
    ...                                  # save_kwargs 为空
elif downstream_block_uuids is not None: # 改下游关系
    ...                                  # save_kwargs 为空
else:                                    # 只改 block 自身属性
    save_kwargs['block_uuid'] = block.uuid  # L2120: 塞入 block_uuid！

# L2138-L2143: 最终调用
self.save(
    block_type=block.type,
    extension_uuid=extension_uuid,
    widget=widget,
    **save_kwargs,                       # 可能包含 block_uuid
)
```

**结论**：
- **改关系（upstream/callback/conditional/downstream）**：`save_kwargs` 为空，`save()` 无 `block_uuid` → 全量写盘
- **只改自身属性（color/configuration/name 等）**：`save_kwargs` 含 `block_uuid` → 走精确保存分支

### 3.3 其他同步保存入口

| 方法 | save 调用 | 传 block_uuid？ | 写盘方式 |
|-----|----------|----------------|---------|
| `Pipeline.add_block()` L1879 | `save()` | ❌ | 全量 |
| `Pipeline.delete_block()` L2320 | `save()` | ❌ | 全量 |
| `Pipeline.update_block()` L2138 | `save(**save_kwargs)` | 视条件 ✅/❌ | 精确 or 全量 |
| `Pipeline.rename_block()` L2206 | `save()` | ❌ | 全量 |
| `Pipeline.update_global_variable()` L2217 | `save()` | ❌ | 全量 |
| `Pipeline.delete_global_variable()` L2221 | `save()` | ❌ | 全量 |

---

## 四、save() / save_async() 参数分支详解

### 4.1 函数签名

```python
# save() 同步版（pipeline.py L2329-L2398）
def save(self, block_type=None, block_uuid=None, extension_uuid=None, widget=False)

# save_async() 异步版（pipeline.py L2419-L2495）
async def save_async(self, block_type=None, block_uuid=None, extension_uuid=None, widget=False)
```

两者签名相同，但内部逻辑有差异。

### 4.2 save() 的 block_uuid 分支（同步精确保存）

当 `block_uuid is not None` 时（L2342-L2368）：

```python
if block_uuid is not None:
    current_pipeline = Pipeline(self.uuid, repo_path=self.repo_path)  # 同步从磁盘加载
    block = self.get_block(block_uuid, block_type=block_type,
                           extension_uuid=extension_uuid, widget=widget)

    if widget:
        current_pipeline.widgets_by_uuid[block_uuid] = block
    elif BlockType.EXTENSION == block.type:        # ← 读 block 实例的 type
        self.extensions[extension_uuid]['blocks_by_uuid'][block_uuid] = block
    elif BlockType.CALLBACK == block.type:          # ← 读 block 实例的 type
        current_pipeline.callbacks_by_uuid[block_uuid] = block
    elif BlockType.CONDITIONAL == block.type:       # ← 读 block 实例的 type
        current_pipeline.conditionals_by_uuid[block_uuid] = block
    else:
        current_pipeline.blocks_by_uuid[block_uuid] = block

    pipeline_dict = current_pipeline.to_dict(
        include_execution_framework=True,
        include_extensions=True,
    )
```

**实际调用场景**：`Pipeline.update_block()` 在只改 block 自身属性时传入 `block_uuid=block.uuid`，走这个分支。

### 4.3 save_async() 的 block_uuid 分支（异步精确保存）

当 `block_uuid is not None` 时（L2428-L2445）：

```python
if block_uuid is not None:
    current_pipeline = await Pipeline.get_async(self.uuid, self.repo_path)  # 异步从磁盘加载
    block = self.get_block(block_uuid, extension_uuid=extension_uuid, widget=widget)

    if widget:
        current_pipeline.widgets_by_uuid[block_uuid] = block
    elif extension_uuid:                            # ← 读函数参数！
        self.extensions[extension_uuid]['blocks_by_uuid'][block_uuid] = block
    elif BlockType.CALLBACK == block_type:           # ← 读函数参数！
        current_pipeline.callbacks_by_uuid[block_uuid] = block
    elif BlockType.CONDITIONAL == block_type:        # ← 读函数参数！
        current_pipeline.conditionals_by_uuid[block_uuid] = block
    else:
        current_pipeline.blocks_by_uuid[block_uuid] = block

    pipeline_dict = current_pipeline.to_dict(include_extensions=True)
```

**当前状态**：没有任何实际调用传入 `block_uuid` 走这个异步分支。入口 A 全部走无 `block_uuid` 的全量写盘；入口 B 走同步 `save()` 而非 `save_async()`。

### 4.4 无 block_uuid 分支（全量写盘）

```python
else:
    # save() 版：
    pipeline_dict = self.to_dict(exclude_data_integration=True,
                                  include_execution_framework=True,
                                  include_extensions=True)
    # save_async() 版：
    pipeline_dict = self.to_dict(exclude_data_integration=True,
                                  include_extensions=True)
```

直接序列化 `self`，不做磁盘读取合并。

---

## 五、发现的问题

### ⚠️ 问题 1：链路 A 的 save_async 调用③ 传了 block_type 和 widget，但没传 block_uuid

`Pipeline.update()` L1532-L1536：

```python
await self.save_async(block_type=block.type, widget=widget)
```

- `block_uuid=None` → 走 else 分支
- `block_type` 和 `widget` 在 else 分支中**完全不使用**
- 等价于 `await self.save_async()`，两个参数是**死参数**

不会出 bug（全量保存本来就正确），但容易误导读者以为走了精确保存逻辑。

---

### ⚠️ 问题 2：save() 和 save_async() 对 extension block 的判断逻辑不一致

**save() 同步版 L2353**：
```python
elif BlockType.EXTENSION == block.type:    # 读 block 实例的 type
```

**save_async() 异步版 L2433**：
```python
elif extension_uuid:                       # 读函数参数
```

| 场景 | save() 判断 | save_async() 判断 | 一致？ |
|------|------------|-------------------|--------|
| Extension block，extension_uuid 有值 | `BlockType.EXTENSION == block.type` → True | `extension_uuid` truthy → True | ✅ |
| Extension block，extension_uuid 漏传 | `BlockType.EXTENSION == block.type` → True | `extension_uuid` falsy → 错误分支 | ❌ |

当前同步入口 B 中 `extension_uuid` 总是从 `block.extension_uuid` 取值，不会漏传，所以不会触发此问题。但 `save_async()` 的 `block_uuid` 分支一旦被启用（目前是死代码），就可能埋坑。

---

### ⚠️ 问题 3：save() 用 block.type 判断，save_async() 用 block_type 参数判断 callback/conditional

**save()**：`BlockType.CALLBACK == block.type`（实例属性，可靠）
**save_async()**：`BlockType.CALLBACK == block_type`（函数参数，依赖调用方正确传值）

两者风格不一致。同步版更安全，异步版在参数传错时会将 block 放入错误 mapping。

---

### ⚠️ 问题 4：链路 A 中每个有变化的 block 都触发一次 save_async()，导致多次全量写盘

`Pipeline.update()` L1532-L1536 的 `should_save_async` 在每个 block 的 for 循环末尾判断，有变化就立即 `save_async()`。由于 `block_uuid=None`，每次都是全量 `self.to_dict()` 写盘。

如果一次编辑保存中有 N 个 block 有变化，metadata.yaml 会被完整写入 N 次（加上元数据变化时的调用②，最多 N+1 次）。幂等但浪费 I/O。

---

### ⚠️ 问题 5：save() 和 save_async() 全量分支的序列化参数不一致

| 参数 | save() 全量分支 | save_async() 全量分支 |
|-----|----------------|---------------------|
| `include_execution_framework` | `True` | **未传**（缺失） |
| `exclude_data_integration` | `True` | `True` |
| `include_extensions` | `True` | `True` |

`save_async()` 全量分支**缺少** `include_execution_framework=True`。这意味着异步全量保存输出的 YAML 可能比同步版少一个 `execution_framework` 字段。

---

### ⚠️ 问题 6：链路 A 全量写盘 vs 链路 B 精确保存的并发冲突

链路 B 的 `save(block_uuid=xxx)` 走"先读磁盘再合并"精确保存，保护了链路 B 之间的并发。但链路 A 的 `save_async()`（无 block_uuid）直接 `self.to_dict()` 全量写盘，**会覆盖磁盘上链路 B 刚写入的更新**。

**场景**：
1. 用户 A 通过编辑页面保存（链路 A），self 内存中有 block1 和 block2 的最新内容
2. 用户 B 通过 API 单独更新了 block3（链路 B），磁盘上 block3 已更新
3. 用户 A 的 `save_async()` 执行 `self.to_dict()` → self 中没有 block3 的最新内容
4. **结果**：用户 B 对 block3 的更新被用户 A 的全量写盘覆盖

链路 B 的精确保存只保护了 B-B 并发，无法防御 A-B 并发。

---

## 六、各 save 调用点参数对照表

| 调用位置 | 方法 | block_type | block_uuid | extension_uuid | widget | 写盘方式 |
|---------|------|-----------|-----------|---------------|--------|---------|
| L1277 (rename) | `save_async()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L1351 (元数据变化) | `save_async()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L1533 (block内容变化) | `save_async(block_type=, widget=)` | ✅ | ❌ | ❌ | ✅ | 全量（死参数） |
| L1879 (add_block) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2138 (update_block-改关系) | `save(block_type=, ext_uuid=, widget=)` | ✅ | ❌ | ✅ | ✅ | 全量 |
| L2138 (update_block-改属性) | `save(block_type=, block_uuid=, ext_uuid=, widget=)` | ✅ | ✅ | ✅ | ✅ | **精确保存** |
| L2206 (rename_block) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2217 (update_global_variable) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2221 (delete_global_variable) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2320 (delete_block) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |

**修正上一版结论**：`save()` 的 `block_uuid` 分支**不是死代码**。`Pipeline.update_block()` 在只改 block 自身属性时，会通过 `save_kwargs` 传入 `block_uuid`，走"先读磁盘再合并"的精确保存分支。

但 `save_async()` 的 `block_uuid` 分支**确实是死代码**——当前没有任何异步路径传入 `block_uuid`。

---

## 七、save() vs save_async() 参数处理差异汇总

| 方面 | save() 同步版 | save_async() 异步版 |
|-----|-------------|-------------------|
| Extension 判断 | `BlockType.EXTENSION == block.type`（读实例） | `elif extension_uuid:`（读参数） |
| Callback 判断 | `BlockType.CALLBACK == block.type`（读实例） | `BlockType.CALLBACK == block_type`（读参数） |
| Conditional 判断 | `BlockType.CONDITIONAL == block.type`（读实例） | `BlockType.CONDITIONAL == block_type`（读参数） |
| 磁盘读取 | `Pipeline(self.uuid, ...)` 同步构造 | `await Pipeline.get_async(...)` 异步构造 |
| YAML 校验 | 无 | 有（写 .test 临时文件校验合法性） |
| 时间戳防碰撞 | `time.sleep(0.0005)` | 无 |
| 全量序列化参数 | `include_execution_framework=True` | **缺少** `include_execution_framework` |
| data_integration 写入 | 同步 `open/write` | 异步 `aiofiles.open/write` |
| block_uuid 分支是否活跃 | ✅ 活跃（update_block 改属性时触发） | ❌ 死代码（无异步调用传入 block_uuid） |

**最关键的差异**：`save()` 的 `block_uuid` 分支活跃且有防并发保护，`save_async()` 的 `block_uuid` 分支是死代码，所有异步保存都是全量覆盖。

---

## 八、对上一版分析的修正

上一版得出"所有保存调用都全量写盘"的结论，这是**不准确的**。修正如下：

1. **`Pipeline.update_block()` 在只改 block 自身属性时，确实传入 `block_uuid`，走 `save()` 的精确保存分支**。这不是死代码。

2. **`save_async()` 的 `block_uuid` 分支确实是死代码**，因为异步编辑保存（入口 A）从不传 `block_uuid`。

3. 上版遗漏了入口 B（同步单区块保存）的完整分析，只关注了 `Pipeline.update()` 内部的调用点，没注意到 `Pipeline.update_block()` 是另一个独立入口，且它**确实使用了 `block_uuid` 精确保存**。

4. "先读后合并"防并发机制在同步路径中**确实生效**，只是在异步路径中不生效。

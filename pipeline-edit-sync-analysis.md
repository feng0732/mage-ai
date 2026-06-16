# Pipeline 编辑同步保存阶段调用参数详解

## 一、保存入口总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│  入口 A：异步编辑保存（前端编辑页面 → PUT ?update_content=true）           │
│                                                                          │
│  前端 savePipelineContent()                                              │
│  → PUT /api/pipelines/:uuid?update_content=true                         │
│  → PipelineResource.update()                                            │
│  → Pipeline.update(data, update_content=True)                           │
│      │                                                                    │
│      ├─ 阶段 A：元数据变化 → save_async()  ← 异步全量保存                │
│      │                                                                    │
│      └─ 阶段 B：遍历每个 block                                           │
│          │                                                                │
│          ├─ block.update_content_async(content)  ← 写 .py/.sql 块内容   │
│          │   └─ __update_pipeline_block(widget=widget)  ← L3208        │
│          │       └─ pipeline.update_block(block, widget=widget)         │
│          │           ├─ save_kwargs = {block_uuid: block.uuid}         │
│          │           └─ save(block_type=, block_uuid=, ext_uuid=,       │
│          │                          widget=)  ← 同步精确保存！           │
│          │               ↑ 这是支线 1：先读磁盘→只替换该 block→写回      │
│          │                                                                │
│          ├─ block.save_outputs_async()       ← 写变量输出                │
│          ├─ block.update(has_callback/color) ← 更新内存属性              │
│          ├─ block.configuration = ...           ← 设 should_save_async=T│
│          ├─ block.update(name/upstream_blocks)  ← 设 should_save_async=T│
│          ├─ block 重命名                           ← 设 should_save_async=T│
│          │                                                                │
│          └─ if should_save_async:                                          │
│              save_async(block_type=, widget=)  ← 异步全量保存！          │
│                  ↑ 这是支线 2：全量 self.to_dict() 覆盖写盘              │
│                                                                          │
│  结果：每次块内容变化都会触发一次同步精确保存，元数据变化再追加一次全量    │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  入口 B：同步单区块保存（Block 属性/关系变更 → 非编辑页面）                │
│                                                                          │
│  BlockResource.update() / Block.update() / Block.__update_pipeline_block()│
│  → Pipeline.update_block(block, ...)                                     │
│      ├─ 有 upstream/callback/conditional/downstream 参数                 │
│      │   → save_kwargs 为空 → save(block_type=, ext_uuid=, widget=)     │
│      │                         ← 无 block_uuid，全量写盘                 │
│      └─ else（只改 block 自身属性）                                      │
│          → save_kwargs = {block_uuid: block.uuid}                       │
│          → save(block_type=, block_uuid=, ext_uuid=, widget=)           │
│            ← 有 block_uuid，同步精确保存                                 │
│                                                                          │
│  Pipeline.add_block() / delete_block()                                   │
│  → save()  ← 无 block_uuid，全量写盘                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 二、入口 A：异步编辑保存 —— 两条保存支线

### 2.1 前端到后端的参数传递

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

### 2.3 模型层：Pipeline.update(data, update_content=True)

`Pipeline.update`（`pipeline.py` L1246-L1589）内部有 **3 次** `save_async` 调用，以及 **N 次**隐式的 `save` 调用（通过 `__update_pipeline_block` 支线）。

#### 支线 1：块内容写入后触发的同步精确保存（关键！）

**关键发现**：`block.update_content_async()` 内部会调用 `__update_pipeline_block()` → `pipeline.update_block()` → `save(block_uuid=block.uuid)`，这是一条**隐式同步保存支线**，之前被遗漏了。

调用链：

```
Pipeline.update() L1439
  └─ block.update_content_async(content, widget=widget)
      ├─ L3207: await self.file.update_content_async(content)
      │    ← 写块内容文件（.py/.sql/.yaml 等）
      └─ L3208: self.__update_pipeline_block(widget=widget)
          └─ L4211: self.pipeline.update_block(self, widget=widget)
              ├─ L2018: save_kwargs = dict()
              ├─ L2022-L2115: 检查 upstream/callback/conditional/downstream
              │    都为 None（调用时没传这些参数）
              ├─ L2120: else 分支 → save_kwargs['block_uuid'] = block.uuid
              └─ L2138-L2143: self.save(
                      block_type=block.type,
                      extension_uuid=extension_uuid,
                      widget=widget,
                      **save_kwargs,  # {block_uuid: block.uuid}
                  )
```

**`save(block_uuid=block.uuid)` 的执行逻辑**（`pipeline.py` L2342-L2368）：

```python
if block_uuid is not None:
    # 1. 从磁盘同步加载 current_pipeline
    current_pipeline = Pipeline(self.uuid, repo_path=self.repo_path)
    
    # 2. 从 self 内存中取出目标 block
    block = self.get_block(block_uuid, block_type=block_type,
                           extension_uuid=extension_uuid, widget=widget)
    
    # 3. 按类型把 block 塞进 current_pipeline
    if widget:
        current_pipeline.widgets_by_uuid[block_uuid] = block
    elif BlockType.EXTENSION == block.type:      # ← 读 block 实例的 type
        self.extensions[extension_uuid]['blocks_by_uuid'][block_uuid] = block
    elif BlockType.CALLBACK == block.type:        # ← 读 block 实例的 type
        current_pipeline.callbacks_by_uuid[block_uuid] = block
    elif BlockType.CONDITIONAL == block.type:     # ← 读 block 实例的 type
        current_pipeline.conditionals_by_uuid[block_uuid] = block
    else:
        current_pipeline.blocks_by_uuid[block_uuid] = block
    
    # 4. 序列化 current_pipeline（非 self）写盘
    pipeline_dict = current_pipeline.to_dict(
        include_execution_framework=True,
        include_extensions=True,
    )
```

**这是同步精确保存**：先读磁盘 → 只替换目标 block → 写回。每个块内容变化都会触发一次。

#### 支线 2：元数据变化后触发的异步全量保存

在 `Pipeline.update()` 的 for 循环末尾（L1532-L1536）：

```python
if should_save_async:
    await self.save_async(
        block_type=block.type,
        widget=widget,
    )
```

`should_save_async` 在以下情况被设为 `True`：
- `configuration` 变化（L1473）
- widget 的 `name` 或 `upstream_blocks` 变化（L1489）
- block 重命名（L1530）

**`save_async(block_type=block.type, widget=widget)` 的执行逻辑**：
- `block_uuid=None`（没传）→ 走 `else` 分支（L2446-L2453）
- `block_type` 和 `widget` 参数在 else 分支中**完全不使用**（死参数）
- 直接 `self.to_dict()` 全量序列化后写盘

**这是异步全量保存**：全量覆盖 metadata.yaml。仅在元数据变化时追加触发。

#### 调用 ① 和 ②：元数据变化时的异步全量保存

- **调用 ① L1277**：pipeline 重命名时 `await self.save_async()`
- **调用 ② L1351**：extensions/tags/description/type/executor_type/retry_config/settings/block 顺序等任一变化时 `await self.save_async()`

都是无参数调用，走全量写盘分支。

### 2.4 两条支线的时序关系

对**每个 block**，如果**内容有变化**，执行顺序为：

```
1. block.update_content_async(content)  ← 写 .py/.sql 块内容文件（异步等待）
   └─ __update_pipeline_block()
       └─ pipeline.update_block(block)
           └─ save(block_uuid=xxx)      ← 同步精确保存 metadata.yaml
                                            ↑ 先读磁盘→只替换该 block→写回
2. block.save_outputs_async()            ← 写变量输出（异步等待）
3. block.update(has_callback/color)      ← 更新内存属性（同步）
4. block.configuration = ...             ← 设 should_save_async=True
5. block.update(name/upstream_blocks)    ← 设 should_save_async=True
6. block 重命名                           ← 设 should_save_async=True
7. if should_save_async:
     save_async(block_type=, widget=)    ← 异步全量保存 metadata.yaml
                                            ↑ 全量 self.to_dict() 覆盖
```

如果**内容无变化但元数据有变化**，则跳过步骤 1，只执行 3-7。

**修正上一版结论**："异步编辑保存只全量写盘"的说法**错误**。实际上：
- ✅ 每次块内容变化都会触发一次**同步精确保存**（`save(block_uuid=xxx)`）
- ✅ 如果元数据也变化了，再追加一次**异步全量保存**（`save_async()`）

---

## 三、入口 B：同步单区块保存 —— 关键路径

### 3.1 触发路径

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

**实际调用场景**：
- 入口 A：`block.update_content_async()` → `__update_pipeline_block()` → `update_block()` → `save(block_uuid=block.uuid)`
- 入口 B：`update_block()` 只改自身属性时 → `save(block_uuid=block.uuid)`

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

## 五、各 save 调用点参数对照表

| 调用位置 | 方法 | block_type | block_uuid | extension_uuid | widget | 写盘方式 |
|---------|------|-----------|-----------|---------------|--------|---------|
| L1277 (rename) | `save_async()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L1351 (元数据变化) | `save_async()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L1533 (block元数据变化) | `save_async(block_type=, widget=)` | ✅ | ❌ | ❌ | ✅ | 全量（死参数） |
| L2138 (update_block-改关系) | `save(block_type=, ext_uuid=, widget=)` | ✅ | ❌ | ✅ | ✅ | 全量 |
| L2138 (update_block-改属性) | `save(block_type=, block_uuid=, ext_uuid=, widget=)` | ✅ | ✅ | ✅ | ✅ | **精确保存** |
| L4211 (update_content_async→__update_pipeline_block) | `save(block_type=, block_uuid=, ext_uuid=, widget=)` | ✅ | ✅ | ✅ | ✅ | **精确保存** |
| L1879 (add_block) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2206 (rename_block) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2217 (update_global_variable) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2221 (delete_global_variable) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |
| L2320 (delete_block) | `save()` | ❌ | ❌ | ❌ | ❌ | 全量 |

**修正上一版结论**：
- `save()` 的 `block_uuid` 分支**不是死代码**，它在**两个场景**中被调用：
  1. 入口 A：`block.update_content_async()` → `__update_pipeline_block()` → `update_block()` → `save(block_uuid=xxx)`
  2. 入口 B：`update_block()` 只改自身属性时 → `save(block_uuid=xxx)`
- `save_async()` 的 `block_uuid` 分支**确实是死代码**——当前没有任何异步路径传入 `block_uuid`

---

## 六、发现的问题

### ⚠️ 问题 1：入口 A 的 save_async 调用③ 传了 block_type 和 widget，但没传 block_uuid

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

当前同步入口中 `extension_uuid` 总是从 `block.extension_uuid` 取值，不会漏传，所以不会触发此问题。但 `save_async()` 的 `block_uuid` 分支一旦被启用（目前是死代码），就可能埋坑。

---

### ⚠️ 问题 3：save() 用 block.type 判断，save_async() 用 block_type 参数判断 callback/conditional

**save()**：`BlockType.CALLBACK == block.type`（实例属性，可靠）
**save_async()**：`BlockType.CALLBACK == block_type`（函数参数，依赖调用方正确传值）

两者风格不一致。同步版更安全，异步版在参数传错时会将 block 放入错误 mapping。

---

### ⚠️ 问题 4：入口 A 中每个有元数据变化的 block 都触发一次 save_async()，导致多次全量写盘

`Pipeline.update()` L1532-L1536 的 `should_save_async` 在每个 block 的 for 循环末尾判断，有变化就立即 `save_async()`。由于 `block_uuid=None`，每次都是全量 `self.to_dict()` 写盘。

如果一次编辑保存中有 N 个 block 的元数据有变化，metadata.yaml 会被完整写入 N 次（加上阶段 A 的调用②，最多 N+1 次）。幂等但浪费 I/O。

---

### ⚠️ 问题 5：save() 和 save_async() 全量分支的序列化参数不一致

| 参数 | save() 全量分支 | save_async() 全量分支 |
|-----|----------------|---------------------|
| `include_execution_framework` | `True` | **未传**（缺失） |
| `exclude_data_integration` | `True` | `True` |
| `include_extensions` | `True` | `True` |

`save_async()` 全量分支**缺少** `include_execution_framework=True`。这意味着异步全量保存输出的 YAML 可能比同步版少一个 `execution_framework` 字段。

---

### ⚠️ 问题 6：同步精确保存 vs 异步全量保存的并发冲突

同步精确保存（`save(block_uuid=xxx)`）走"先读磁盘再合并"，保护了同步调用之间的并发。但异步全量保存（`save_async()`）直接 `self.to_dict()` 全量写盘，**会覆盖磁盘上同步精确保存刚写入的更新**。

**场景**：
1. 用户 A 编辑 block1 内容（触发 `save(block_uuid=block1)` 精确保存）
2. 用户 B 编辑 block2 内容（触发 `save(block_uuid=block2)` 精确保存）
3. 用户 A 的 block1 元数据也变化了（触发 `save_async()` 全量保存）
4. 此时 `self` 内存中只有 block1 的最新元数据，没有 block2 的最新元数据
5. **结果**：用户 B 对 block2 的元数据更新被用户 A 的全量写盘覆盖

同步精确保存只保护了 block 内容的并发，无法防御异步全量保存对元数据的覆盖。

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
| block_uuid 分支是否活跃 | ✅ 活跃（入口 A 内容更新 + 入口 B 属性更新） | ❌ 死代码（无异步调用传入 block_uuid） |

---

## 八、对上一版分析的修正

上一版得出"异步编辑保存只全量写盘"的结论，这是**严重错误**。修正如下：

1. **入口 A（异步编辑保存）中存在两条保存支线**：
   - **支线 1**（每次内容变化触发）：`block.update_content_async()` → `__update_pipeline_block()` → `update_block()` → `save(block_uuid=xxx)` → **同步精确保存**（先读磁盘再合并）
   - **支线 2**（元数据变化时追加）：`save_async(block_type=, widget=)` → **异步全量保存**（全量 self.to_dict()）

2. **`save()` 的 `block_uuid` 分支不仅在入口 B 中活跃，在入口 A 中也活跃**。每次块内容变化都会触发一次同步精确保存。

3. **`__update_pipeline_block()` 是连接块内容更新和元数据保存的关键桥梁**，之前的分析完全遗漏了这条隐式支线。

4. **"先读后合并"防并发机制在入口 A 中也生效**，但只保护块内容更新，不保护元数据更新（元数据更新走异步全量保存，会覆盖并发修改）。

---

## 九、关键文件速查表

| 文件 | 仓库路径 | 角色 |
|------|---------|------|
| `edit.tsx` | `mage_ai/frontend/pages/pipelines/[pipeline]/edit.tsx` | 入口 A 发起层：暂存内容、构造 payload、mutation 回调 |
| `PipelineDetail/index.tsx` | `mage_ai/frontend/components/PipelineDetail/index.tsx` | 入口 A 发起层：快捷键 + 自动保存触发 |
| `PipelineResource.py` | `mage_ai/api/resources/PipelineResource.py` | 入口 A 承接层：解析 update_content、注册收尾回调 |
| `pipeline.py` | `mage_ai/data_preparation/models/pipeline.py` | 入口 A 处理层：Pipeline.update + save/save_async 写文件 + 缓存收尾 |
| `BlockResource.py` | `mage_ai/api/resources/BlockResource.py` | 入口 B 承接层：Block.update 属性变更 |
| `block/__init__.py` | `mage_ai/data_preparation/models/block/__init__.py` | 入口 B 发起层：Block.update → Pipeline.update_block + __update_pipeline_block 支线 |
| `StatusFooter/index.tsx` | `mage_ai/frontend/components/PipelineDetail/StatusFooter/index.tsx` | 展示层：已保存/未保存图标与文案 |

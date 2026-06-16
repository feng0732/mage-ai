# Block 模板系统异常兜底——代码实现核准

> **核准方法**：源码阅读 + 运行时实验验证。
> **Jinja2 版本依据**：项目 requirements.txt 声明 `Jinja2==3.1.3`（[requirements.txt L5](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/requirements.txt#L5)），运行时实际安装 `Jinja2==3.1.6`。两者同属 3.1.x 系列且异常继承链一致（`TemplateNotFound → OSError → LookupError → TemplateError → Exception`），下文分析对两个版本均成立。

---

## 一、前置事实：Jinja2 异常类型链

**实验验证**（Jinja2 3.1.6 + Python 3.14）：

```
jinja2.TemplateNotFound
  → OSError
    → LookupError
      → jinja2.TemplateError
        → Exception
```

**核准结论**：

| 问题 | 答案 |
|------|------|
| `TemplateNotFound` 是 `FileNotFoundError` 的子类吗？ | **否** |
| `except FileNotFoundError` 能捕获 `TemplateNotFound` 吗？ | **否**（已实验确认） |
| `except OSError` 能捕获 `TemplateNotFound` 吗？ | **是**（`TemplateNotFound` 继承自 `OSError`） |
| `except Exception` 能捕获 `TemplateNotFound` 吗？ | **是** |

---

## 二、三层缺失边界：目标模板、默认模板、父模板

### 2.1 概念定义

| 层级 | 定义 | 举例 |
|------|------|------|
| **目标模板** | 调用方请求的具体模板文件 | `data_loaders/s3.py`、`sensors/snowflake.py` |
| **默认模板** | 目标模板缺失时回退的兜底模板 | `data_loaders/default.jinja`、`sensors/default.py` |
| **父模板** | Jinja2 `{% extends %}` 继承链中的上游模板 | `testable.jinja`、`data_loaders/default.jinja`、`transformers/default.jinja` |

### 2.2 完整继承链图谱

通过扫描所有 `.jinja` 文件的 `{% extends %}` 语句，得到以下继承链：

```
testable.jinja                          ← 根模板（无 extends）
├── data_loaders/default.jinja          ← 也作为 data_loaders 下 .py 子模板的父模板
│   ├── data_loaders/s3.py              ← 目标模板
│   ├── data_loaders/snowflake.py
│   ├── data_loaders/bigquery.py
│   └── ... (22 个 .py 文件)
│   └── data_loaders/deltalake/default.jinja
├── data_exporters/default.jinja        ← 也作为 data_exporters 下子模板的父模板
│   ├── data_exporters/deltalake/default.jinja
│   └── data_exporters/orchestration/triggers/default.jinja
├── data_loaders/pyspark/default.jinja
├── transformers/default.jinja          ← 也作为 transformer 子模板的父模板
│   ├── transformers/transformer_actions/action.jinja
│   ├── transformers/suggestion_fmt.jinja
│   └── transformers/data_warehouse_transformer.jinja
├── transformers/default_pyspark.jinja
├── custom/python/default.jinja

callbacks/base.jinja                    ← 独立根模板（无 extends）
└── callbacks/orchestration/triggers/default.jinja

（无 extends 的独立模板）
├── conditionals/base.jinja             ← 无继承，自身即根
├── sensors/default.py                  ← 纯 Python，无 Jinja2 继承
├── callbacks/default.py                ← 纯 Python，无 Jinja2 继承
└── transformers/default_streaming.jinja ← 无 extends，自身即根
```

### 2.3 三层缺失的异常行为对照

| 缺失层级 | 发生时机 | 抛出异常 | 能否被现有兜底捕获 | 实际影响 |
|---------|---------|---------|-----------------|---------|
| **目标模板缺失** | `get_template('data_loaders/s3.py')` | `TemplateNotFound` | #2、#3 分支通过 `template_exists()` 预检查绕过；其余分支直接穿透 | #2/#3 回退成功；其余 500 |
| **默认模板缺失** | `get_template('data_loaders/default.jinja')` | `TemplateNotFound` | `template_exists()` 只检查目标模板，不检查默认模板；无第二级兜底 | 即使 #2/#3 也 500 |
| **父模板缺失** | 渲染 `data_loaders/s3.py` 时 Jinja2 解析 `{% extends "data_loaders/default.jinja" %}` | `TemplateNotFound` | 无任何代码防御 | **隐性 500**：目标模板和默认模板都存在，但父模板缺失同样导致渲染失败 |

**关键发现**：父模板缺失是**隐性风险**——当前所有异常兜底逻辑（`template_exists()` 预检查、try-except）都只关注目标模板和默认模板是否存在，完全忽略了 Jinja2 `{% extends %}` 渲染时对父模板的依赖。即使目标模板文件完好，只要其父模板缺失，`.render()` 调用仍会抛出 `TemplateNotFound`。

### 2.4 各模板的三层依赖关系表

| 目标模板 | 默认模板（回退目标） | 父模板（extends 链） | 缺失任一层即失败 |
|---------|-------------------|---------------------|----------------|
| `data_loaders/s3.py` | `data_loaders/default.jinja` | `data_loaders/default.jinja` → `testable.jinja` | ✅ |
| `data_loaders/deltalake/default.jinja` | `data_loaders/default.jinja` | `data_loaders/default.jinja` → `testable.jinja` | ✅ |
| `data_loaders/pyspark/default.jinja` | （自身即默认） | `testable.jinja` | ✅ |
| `data_exporters/deltalake/default.jinja` | `data_exporters/default.jinja` | `data_exporters/default.jinja`（无 extends） | ✅ |
| `transformers/transformer_actions/action.jinja` | `transformers/default.jinja`（意图回退但无效） | `transformers/default.jinja` → `testable.jinja` | ✅ |
| `transformers/data_warehouse_transformer.jinja` | （无回退） | `transformers/default.jinja` → `testable.jinja` | ✅ |
| `transformers/suggestion_fmt.jinja` | （无回退） | `transformers/default.jinja` → `testable.jinja` | ✅ |
| `transformers/default.jinja` | （自身即默认） | `testable.jinja` | ✅ |
| `transformers/default_pyspark.jinja` | （自身即默认） | `testable.jinja` | ✅ |
| `custom/python/default.jinja` | （自身即默认） | `testable.jinja` | ✅ |
| `callbacks/orchestration/triggers/default.jinja` | `callbacks/base.jinja` | `callbacks/base.jinja`（无 extends） | ✅ |
| `sensors/snowflake.py` | `sensors/default.py` | 无（纯 Python 不使用 extends） | 仅缺目标/默认 |
| `callbacks/default.py` | （自身即默认） | 无（纯 Python） | 仅缺自身 |
| `conditionals/base.jinja` | （自身即默认） | 无（无 extends） | 仅缺自身 |
| `data_exporters/default.jinja` | （自身即默认） | 无（无 extends，不继承 testable） | 仅缺自身 |

**规律总结**：
- 所有 `.jinja` 模板中，`data_exporters/default.jinja`、`conditionals/base.jinja`、`callbacks/base.jinja`、`transformers/default_streaming.jinja` 没有继承关系，缺失自身即失败，无隐性依赖
- 纯 `.py` 模板（sensors、callbacks）没有 Jinja2 继承，只有目标/默认两层
- 其余模板都至少依赖 `testable.jinja` 这个根模板

---

## 三、逐分支核准：模板文件缺失时的真实行为

### 3.1 核准总表

| # | 分支函数 | 代码防御手段 | **核准结论** | 真实异常类型 |
|---|---------|------------|-------------|------------|
| 1 | `config['template_path']` 直取 | 无 | **直接抛错** ❌ | `TemplateNotFound` |
| 2 | `__fetch_data_loader_templates` | `template_exists()` 预检查 | **真实回退** ✅ | 不抛异常（路径已验证） |
| 3 | `__fetch_data_exporter_templates` | `template_exists()` 预检查 | **真实回退** ✅ | 不抛异常（路径已验证） |
| 4 | `__fetch_transformer_templates` 默认分支 | 无 | **直接抛错** ❌ | `TemplateNotFound` |
| 5 | `__fetch_transformer_data_warehouse_template` | 无（+ DataSource 映射检查） | **直接抛错** ❌ | `TemplateNotFound` 或 `ValueError` |
| 6 | `__fetch_transformer_action_template` | `except FileNotFoundError` | **假回退** ⚠️ | `TemplateNotFound` 穿透 |
| 7 | `__fetch_sensor_templates` | `except ValueError`（仅枚举校验） | **直接抛错** ❌ | `TemplateNotFound` |
| 8 | `__fetch_custom_templates` | 无 | **直接抛错** ❌ | `TemplateNotFound` |
| 9 | `__fetch_callback_templates` | 无 | **直接抛错** ❌ | `TemplateNotFound` |
| 10 | `__fetch_conditional_templates` | 无 | **直接抛错** ❌ | `TemplateNotFound` |
| 11 | `build_template_from_suggestion` | 无 | **直接抛错** ❌ | `TemplateNotFound` |
| 12 | `data_integrations.render_template` | 无 | **直接抛错** ❌ | `TemplateNotFound` |

**结论**：12 个分支中，**仅 2 个能真实回退**（#2、#3），**9 个直接抛错**，**1 个假回退**（#6）。

---

### 3.2 逐分支核准详解

#### #1 config['template_path'] 直取

**位置**：[template.py L66-L87](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L66-L87)

```python
if 'template_path' in config:
    if TEMPLATE_TYPE_DATA_INTEGRATION == config.get('template_type'):
        return render_template(...)
    return (
        template_env.get_template(config['template_path']).render(
            **template_variables_to_render,
        )
    )
```

- **防御手段**：无
- **核准结论**：直接抛错 ❌
- **三层缺失影响**：目标模板缺失 → `TemplateNotFound`；父模板缺失 → `.render()` 时同样 `TemplateNotFound`

---

#### #2 __fetch_data_loader_templates ✅ 真实回退

**位置**：[template.py L159-L167](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L159-L167)

```python
data_source_template = template_folder + '/' + f'{data_source.lower()}.{file_extension}'
if template_exists(data_source_template):
    template_path = data_source_template
else:
    template_path = default_template
```

**核准过程**：

1. `template_exists()` 的实现（[utils.py L65-L79](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/utils.py#L65-L79)）：
   ```python
   def template_exists(template_path: str) -> bool:
       template_path = os.path.join(os.path.dirname(__file__), template_path)
       return os.path.exists(template_path)
   ```
   基准路径是 `os.path.dirname(__file__)`，即 `mage_ai/data_preparation/templates/`

2. `template_env` 的初始化（[utils.py L10-L14](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/utils.py#L10-L14)）：
   ```python
   template_env = jinja2.Environment(
       loader=jinja2.FileSystemLoader(os.path.dirname(__file__)),
   )
   ```
   基准路径也是 `os.path.dirname(__file__)`

3. **路径基准完全一致**：已通过实验确认两者对同一相对路径的存在性判断一致。

- **核准结论**：真实回退 ✅
- **回退目标**：`{template_folder}/default.jinja`（受 pipeline_type/language 影响，有 5 种变体）
- **三层缺失边界**：
  - 目标模板缺失 → `template_exists()` 返回 False → 回退到默认模板 ✅
  - 默认模板缺失 → `template_exists()` 不检查默认模板 → `get_template()` 抛 `TemplateNotFound` ❌
  - **父模板缺失** → `template_exists()` 无法感知 → `.render()` 时抛 `TemplateNotFound` ❌

---

#### #3 __fetch_data_exporter_templates ✅ 真实回退

**位置**：[template.py L283-L291](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L283-L291)

代码结构、防御手段、回退逻辑与 #2 完全对称。

- **核准结论**：真实回退 ✅
- **三层缺失边界**：与 #2 相同

---

#### #4 __fetch_transformer_templates 默认分支

**位置**：[template.py L194-L208](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L194-L208)

```python
if pipeline_type == PipelineType.PYSPARK:
    template_path = 'transformers/default_pyspark.jinja'
elif pipeline_type == PipelineType.STREAMING:
    template_path = 'transformers/default_streaming.jinja'
elif language == BlockLanguage.R:
    template_path = 'transformers/r/default.r'
else:
    template_path = 'transformers/default.jinja'
return template_env.get_template(template_path).render(code=existing_code) + '\n'
```

- **防御手段**：无
- **核准结论**：直接抛错 ❌
- **三层缺失边界**：目标=默认，无回退；`default_pyspark.jinja` 和 `default.jinja` 都继承 `testable.jinja`，父模板缺失时同样 500

---

#### #5 __fetch_transformer_data_warehouse_template

**位置**：[template.py L225-L243](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L225-L243)

```python
template = template_env.get_template('transformers/data_warehouse_transformer.jinja')
data_source_handler = MAP_DATASOURCE_TO_HANDLER.get(data_source)
if data_source_handler is None:
    raise ValueError(f'No associated database/warehouse for data source \'{data_source}\'')
```

- **防御手段**：无模板缺失检查；有 DataSource 映射检查
- **两层抛错**：模板缺失 → `TemplateNotFound`；DataSource 映射失败 → `ValueError`
- **核准结论**：直接抛错 ❌
- **父模板依赖**：`data_warehouse_transformer.jinja` → `transformers/default.jinja` → `testable.jinja`，缺任一层即失败

---

#### #6 __fetch_transformer_action_template ⚠️ 假回退（核心发现）

**位置**：[template.py L246-L253](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L246-L253)

```python
def __fetch_transformer_action_template(action_type: ActionType, axis: Axis, existing_code: str):
    try:
        template = template_env.get_template(
            f'transformers/transformer_actions/{axis}/{action_type}.py'
        )
    except FileNotFoundError:
        template = template_env.get_template('transformers/default.jinja')
    return template.render(code=existing_code) + '\n'
```

**核准过程**：

1. `template_env.get_template()` 模板缺失时抛 `jinja2.TemplateNotFound`
2. `TemplateNotFound` 的继承链：`TemplateNotFound → OSError → LookupError → TemplateError → Exception`
3. `FileNotFoundError` 的继承链：`FileNotFoundError → OSError → Exception`
4. 两者有共同祖先 `OSError`，但 `TemplateNotFound` **不是** `FileNotFoundError` 的子类
5. **实验验证**：

```
>>> try: env.get_template('nonexistent.jinja')
... except FileNotFoundError: print('Would fallback')
... except Exception as e: print(f'Escaped as {type(e).__name__}')
...
Escaped as TemplateNotFound  ← 穿透了 except FileNotFoundError
```

**核准结论**：**假回退** ⚠️

- 代码意图：找不到 action_type 模板时回退到 `transformers/default.jinja`
- 实际行为：`except FileNotFoundError` 无法捕获 `TemplateNotFound`，回退逻辑**永远不会执行**
- **修复方式**：将 `except FileNotFoundError` 改为 `except jinja2.TemplateNotFound`（或 `except OSError`，因为 `TemplateNotFound` 是 `OSError` 的子类）

---

#### #7 __fetch_sensor_templates

**位置**：[template.py L301-L314](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L301-L314)

```python
data_source = config.get('data_source')
try:
    _ = DataSource(data_source)
    template_path = f'sensors/{data_source.lower()}.py'
except ValueError:
    template_path = 'sensors/default.py'
return template_env.get_template(template_path).render(...) + '\n'
```

- **核准结论**：直接抛错 ❌
- `ValueError` 捕获的是「枚举非法」场景，不是「模板文件缺失」场景
- sensor 模板是纯 `.py` 文件，**没有 Jinja2 继承链**，所以不存在父模板缺失的隐性风险

---

#### #8 #9 #10 __fetch_custom/callback/conditional_templates

| # | 函数 | 硬编码路径 | 父模板 | 核准结论 |
|---|------|----------|-------|---------|
| 8 | `__fetch_custom_templates` | `custom/python/default.jinja` | `testable.jinja` | 直接抛错 ❌ |
| 9 | `__fetch_callback_templates` | `callbacks/default.py` | 无（纯 Python） | 直接抛错 ❌ |
| 10 | `__fetch_conditional_templates` | `conditionals/base.jinja` | 无（无 extends） | 直接抛错 ❌ |

---

#### #11 build_template_from_suggestion

**位置**：[template.py L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/template.py#L31-L52)

- `suggestion_fmt.jinja` 继承 `transformers/default.jinja` → `testable.jinja`
- **核准结论**：直接抛错 ❌

---

#### #12 data_integrations.render_template

**位置**：[data_integrations/utils.py L116-L126](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/templates/data_integrations/utils.py#L116-L126)

- **核准结论**：直接抛错 ❌
- 上游 `build_integration_module_info()` 的 try-except 处理的是 `mage_integrations` 包导入失败，不是 Jinja2 模板缺失

---

## 四、自定义模板加载失败：调用方处理核准（含 Pipeline 模板）

### 4.1 CustomBlockTemplate.load 的失败返回值

**位置**：[custom_block_template.py L60-L72](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/models/custom_templates/custom_block_template.py#L60-L72)

```python
try:
    custom_template = super().load(config_path_metadata)
    custom_template.template_uuid = template_uuid_use
    custom_template.repo_path = repo_path
    return custom_template
except Exception as err:
    print(f'[WARNING] CustomBlockTemplate.load: {err}')
    # 无 return → 隐式返回 None
```

**核准结论**：任何异常 → 打印 WARNING → 返回 `None`。调用方必须处理 `None`。

### 4.2 CustomPipelineTemplate.load 的失败返回值

**位置**：[custom_pipeline_template.py L55-L67](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/models/custom_templates/custom_pipeline_template.py#L55-L67)

```python
try:
    config_path_metadata = os.path.join(repo_path, uuid_use, METADATA_FILENAME_WITH_EXTENSION)
    custom_template = super().load(config_path_metadata)
    custom_template.template_uuid = template_uuid_use
    custom_template.repo_path = repo_path
    return custom_template
except Exception as err:
    print(f'[WARNING] CustomPipelineTemplate.load: {err}')
    # 无 return → 隐式返回 None
```

**核准结论**：与 Block 模板完全相同的模式——任何异常 → 打印 WARNING → 返回 `None`。

---

### 4.3 调用方核准（含 Pipeline 分支）

#### 调用方 A：BlockResource.create

**位置**：[BlockResource.py L210-L220](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/BlockResource.py#L210-L220)

```python
custom_template = CustomBlockTemplate.load(repo_path, template_uuid=template_uuid)
block = custom_template.create_block(...)        # ← None.create_block()
content = custom_template.load_template_content() # ← None.load_template_content()
```

- **检查 None？** ❌ 否
- **核准结论**：`None.create_block()` 抛 `AttributeError`，无 try-except 包裹 → HTTP 500

---

#### 调用方 B：PipelineResource.create（Pipeline 自定义模板失败链路）

**位置**：[PipelineResource.py L425-L430](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/PipelineResource.py#L425-L430)

```python
if template_uuid:
    custom_template = CustomPipelineTemplate.load(
        repo_path,
        template_uuid=template_uuid,
    )
    pipeline = custom_template.create_pipeline(name)  # ← None.create_pipeline()
```

- **检查 None？** ❌ 否
- **核准结论**：与 BlockResource.create 完全相同的空指针风险。`CustomPipelineTemplate.load()` 返回 `None` 时，`None.create_pipeline(name)` 抛 `AttributeError` → HTTP 500
- **触发场景**：前端通过 Pipeline 模板创建 Pipeline 时，若自定义 Pipeline 模板的 `metadata.yaml` 损坏或缺失

---

#### 调用方 C：CustomTemplateResource.create

**位置**：[CustomTemplateResource.py L70-L102](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/CustomTemplateResource.py#L70-L102)

**Block 模板分支**：
```python
custom_template = CustomBlockTemplate.load(repo_path, template_uuid=template_uuid)
if not custom_template:  # ← ✅ 检查了 None
    custom_template = CustomBlockTemplate(...)
    custom_template.content = fetch_template_source(...)
    custom_template.save()
```

**Pipeline 模板分支**：
```python
elif DIRECTORY_FOR_PIPELINE_TEMPLATES == object_type:
    custom_template = CustomPipelineTemplate.load(
        template_uuid=template_uuid, repo_path=repo_path
    )
    if not custom_template:  # ← ✅ 检查了 None
        pipeline = Pipeline.get(payload.get('pipeline_uuid'), repo_path=repo_path)
        custom_template = CustomPipelineTemplate.create_from_pipeline(
            pipeline, template_uuid, ...
        )
```

- **检查 None？** ✅ 两个分支都检查了
- **核准结论**：正确处理。`None` 被视为"模板不存在"，走创建新模板流程

---

#### 调用方 D：CustomTemplateResource.member

**位置**：[CustomTemplateResource.py L133-L151](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/CustomTemplateResource.py#L133-L151)

**Block 模板分支**：
```python
return self(
    CustomBlockTemplate.load(repo_path, template_uuid=template_uuid),  # ← 可能 None
    user, **kwargs,
)
```

**Pipeline 模板分支**：
```python
return self(
    CustomPipelineTemplate.load(repo_path, template_uuid=template_uuid),  # ← 可能 None
    user, **kwargs,
)
```

- **检查 None？** ❌ 两个分支都未检查
- **核准结论**：与 Block 模板分支相同，`self(None,...)` → 序列化异常 → `except Exception` 转 404（间接兜底）

---

#### 调用方 E：group_and_hydrate_files

**位置**：[custom_templates/utils.py L59-L76](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/models/custom_templates/utils.py#L59-L76)

```python
for template_uuid, _ in groups.items():
    custom_template = custom_template_class.load(get_repo_path(), template_uuid=template_uuid)
    if custom_template:  # ← ✅ 检查了 None
        custom_templates.append(custom_template)
```

此函数同时服务于 Block 模板和 Pipeline 模板（通过 `custom_template_class` 参数区分）。

- **检查 None？** ✅ 是
- **核准结论**：静默跳过，不加入列表，无日志

---

### 4.4 调用方核准汇总（含 Pipeline）

| 调用方 | 模板类型 | 检查 None | 失败后真实行为 | HTTP 状态 |
|--------|---------|----------|--------------|----------|
| [BlockResource.create](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/BlockResource.py#L212) | Block | ❌ | `None.create_block()` → `AttributeError` | 500 |
| [PipelineResource.create](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/PipelineResource.py#L430) | Pipeline | ❌ | `None.create_pipeline()` → `AttributeError` | 500 |
| [CustomTemplateResource.create](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/CustomTemplateResource.py#L70) | Block | ✅ | 当作不存在 → 创建新模板 | 200 |
| [CustomTemplateResource.create](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/CustomTemplateResource.py#L98) | Pipeline | ✅ | 当作不存在 → 从 Pipeline 创建 | 200 |
| [CustomTemplateResource.member](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/CustomTemplateResource.py#L136) | Block | ❌（间接兜底） | `self(None,...)` → 序列化异常 → 转 404 | 404 |
| [CustomTemplateResource.member](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/CustomTemplateResource.py#L140) | Pipeline | ❌（间接兜底） | `self(None,...)` → 序列化异常 → 转 404 | 404 |
| [group_and_hydrate_files](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/data_preparation/models/custom_templates/utils.py#L74) | 两者 | ✅ | 静默跳过 | N/A |

---

## 五、异常传播链路（核准版）

### 5.1 #6 假回退分支的异常穿透

```
__fetch_transformer_action_template(action_type, axis, existing_code)
    ├─ template_env.get_template(f'transformers/transformer_actions/{axis}/{action_type}.py')
    │   └─ 抛出 jinja2.TemplateNotFound
    ├─ except FileNotFoundError:  ← 无法捕获 TemplateNotFound
    │   (回退代码永远不会执行)
    └─ TemplateNotFound 穿透 → HTTP 500
```

### 5.2 父模板缺失的隐性失败

```
__fetch_data_loader_templates(config)
    ├─ template_exists('data_loaders/s3.py') → True  ← 目标存在
    ├─ template_path = 'data_loaders/s3.py'
    ├─ template_env.get_template('data_loaders/s3.py') → 成功加载模板对象
    ├─ .render(code=existing_code)  ← 渲染时解析 {% extends "data_loaders/default.jinja" %}
    │   ├─ Jinja2 需要加载 data_loaders/default.jinja
    │   │   ├─ 再解析 {% extends "testable.jinja" %}
    │   │   │   ├─ testable.jinja 存在 → 渲染成功 ✅
    │   │   │   └─ testable.jinja 缺失 → TemplateNotFound ❌
    │   │   └─ data_loaders/default.jinja 缺失 → TemplateNotFound ❌
    │   └─ 任何父模板缺失都导致 render() 失败
    └─ 无任何代码防御此场景 → HTTP 500
```

### 5.3 自定义模板 None 穿透链路

```
Custom*.load() 失败 → return None
    ├─ BlockResource.create:    None.create_block()     → AttributeError → 500
    ├─ PipelineResource.create: None.create_pipeline()   → AttributeError → 500
    ├─ CustomTemplateResource.create: if not None → 创建新模板 → 200
    ├─ CustomTemplateResource.member: self(None,...) → 序列化异常 → 转 404
    └─ group_and_hydrate_files: if None → 跳过 → 列表中消失
```

---

## 六、确定结论

### 6.1 哪些分支真实回退

**仅 2 个**：`__fetch_data_loader_templates` 和 `__fetch_data_exporter_templates`。

两者共同的特征：
- 使用 `template_exists()` **先检查再取**，路径基准与 `FileSystemLoader` 一致
- `template_exists()` 返回 `False` 时切换到 `default_template`
- `default_template` 本身缺失时仍抛错（但属于核心资源缺失，属合理行为）
- **父模板缺失时不设防**：`template_exists()` 只检查文件是否存在，不验证 `.render()` 时 Jinja2 继承链的完整性

### 6.2 哪些分支直接抛错

**10 个**，但性质不同：

| 性质 | 分支 | 原因 |
|------|------|------|
| **无防御**（9 个） | #1, #4, #5, #7, #8, #9, #10, #11, #12 | 代码中根本没有 try-except 或预检查 |
| **假回退**（1 个） | #6 `__fetch_transformer_action_template` | 有 try-except 但 `except FileNotFoundError` 无法捕获 `TemplateNotFound` |

### 6.3 三层缺失边界总结

| 缺失层级 | 被 `template_exists()` 检测到？ | 被现有 try-except 兜底？ | 实际影响 |
|---------|------|------|---------|
| 目标模板缺失 | ✅ 是（#2、#3 分支） | ❌ 否（其余分支） | 仅 #2、#3 可回退 |
| 默认模板缺失 | ❌ 否 | ❌ 否 | 所有分支均 500 |
| 父模板缺失 | ❌ 否 | ❌ 否 | 所有使用 extends 的模板均 500（隐性风险） |

### 6.4 Pipeline 自定义模板的空指针风险

[PipelineResource.create](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/PipelineResource.py#L430) 与 [BlockResource.create](file:///d:/fz/0601/solo-dogfeeding/code/320-mage-ai/mage_ai/api/resources/BlockResource.py#L212) 存在**完全相同的空指针风险**：`CustomPipelineTemplate.load()` 返回 `None` 时，直接调用 `None.create_pipeline()` 导致 `AttributeError` → HTTP 500。

### 6.5 Jinja2 版本兼容性

- requirements.txt 声明 `Jinja2==3.1.3`
- 运行时实际安装 `Jinja2==3.1.6`
- 两者同属 3.1.x 系列，`TemplateNotFound` 的继承链（`→ OSError → LookupError → TemplateError → Exception`）在 3.1.x 全系列中一致
- **`except FileNotFoundError` 无法捕获 `TemplateNotFound`** 这一结论对 Jinja2 3.1.3 同样成立（该异常继承关系自 Jinja2 3.0 起未变）

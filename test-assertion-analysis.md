# Mage AI 测试与断言机制深度分析

## 零、代码核准与勘误（本章节基于源码走查精确验证）

> 本章节对前版分析中描述的关键路径进行逐行核准，发现并标注实际代码与设计意图的偏差。

---

### 0.1 自动生成脚本依赖上下文注入：实际**完全失效**（统一结论）

#### 核准结论（最终统一）
`run_process()` → `build_system_prompt()` → `build_test()` 完整调用链中存在 BUG，导致「读取所有依赖源码注入 System Prompt 辅助 AI 生成」的设计意图**从未实际生效**。Claude API 实际收到的上下文仅包含：目标文件源代码 + Few-shot 示范对，**零依赖文件源码**。

#### 完整调用链逐行核准证据

**调用链 1/4：`run_process()` 收集依赖阶段** [write_tests.py#L270-L286](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py#L270-L286)

```python
def run_process(file_path: str):
    # ... 读取目标文件 content ...

    files_imported = extract_mage_ai_imports(content)   # ✅ 第276行：正确提取 mage_ai import 行
    print(f'Files imported: {len(files_imported)}')      # 控制台能看到导入数量

    imported_file_paths = []                             # ⚠️  第279行：空列表初始化
    for import_line in files_imported:
        import_file_path = build_file_path_from_import(import_line)
        print(f'  - {import_file_path}')                 # ❌ 第282行：只打印路径！从未 append 到列表

    prompt = build_prompt(content, file_path)
    system_prompt = build_system_prompt(imported_file_paths)  # ⚠️  第285行：传入空列表！
    result = build_test(prompt, system_prompt=system_prompt)
```

**BUG 根因**：第 282 行**缺少** `imported_file_paths.append(import_file_path)`，导致无论有多少依赖，`imported_file_paths` 恒为 `[]`。

---

**调用链 2/4：`build_system_prompt()` 处理空列表** [write_tests.py#L176-L210](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py#L176-L210)

```python
def build_system_prompt(file_paths: list[str]) -> str:
    documents = []
    for file_path in file_paths:  # ⚠️  file_paths = []，for 循环零次执行
        with open(file_path, 'r') as file:
            code = file.read()
            documents.append(f"<source>...</source><document_content>{code}</document_content>")

    documents_string = '\n'.join(documents)  # ⚠️  documents = []，join 结果 = ""

    return f"""
Here are documents that contain Python code for you to reference:

<documents>
{documents_string}       <!-- ⚠️  最终为空字符串：<documents></documents> -->
</documents>
"""
```

**结果**：System Prompt 中的 `<documents>` 标签内容为空，无任何依赖源码。

---

**调用链 3/4：`build_test()` 传递给 Claude API** [write_tests.py#L213-L242](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py#L213-L242)

```python
def build_test(prompt: str, system_prompt: Optional[str] = None, temperature: float = 1.0):
    message = client.messages.create(
        model=MODEL,
        max_tokens=4096,
        temperature=temperature,
        system=system_prompt,  # ⚠️  传入的是包含空 <documents> 的 system prompt
        messages=[
            {'role': 'user', 'content': [{'text': prompt, 'type': 'text'}]},
            {'role': 'assistant', 'content': '<test>'},  # 前缀填充，引导输出格式
        ],
    )
    return message.content
```

**结果**：Claude API 的 `system` 参数只包含空的文档引用标签，无实际依赖代码。

---

**调用链 4/4：`build_prompt()` 中的用户 Prompt** [write_tests.py#L71-L173](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py#L71-L173)

用户侧 Prompt 仅包含：
1. Few-shot 示范（retry 装饰器代码 + RetryTests 测试代码）
2. 任务说明
3. 目标文件本身的源代码（`<code>{code}</code>`）

**不包含**任何依赖文件的源码。

#### 完整调用链最终结论

```
目标文件内容 + Few-shot 示范
       ↓
build_prompt() → user prompt（仅目标文件）
       ↓
build_system_prompt([]) → system prompt（<documents></documents> 为空）
       ↓
build_test() → Claude API
       ↓
AI 生成的测试仅基于：目标文件本身 + 示范样式
```

#### 次级问题：`build_file_path_from_import()` 的路径映射缺陷（即使修复 append 也会失败）

对 [write_tests.py#L17-L34](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py#L17-L34) 核准：

| import 语句 | 映射结果 | 实际文件 | 结果 |
|------------|---------|---------|------|
| `import mage_ai.shared.retry` | `mage_ai/shared/retry.py` | ✅ 存在 | 正确 |
| `from mage_ai.settings import X` | `mage_ai/settings.py` | ❌ 不存在（是子包目录） | `FileNotFoundError` |
| `from mage_ai.orchestration.db import safe_db_query` | `mage_ai/orchestration/db.py` | ❌ 不存在（是子包目录） | `FileNotFoundError` |

**缺陷根因**：未处理 Python 子包（目录型模块）的 `__init__.py` 约定。

#### 核准总结（统一版）
| 设计意图（预期） | 实际代码行为 | 偏差程度 |
|-----------------|-------------|---------|
| 解析 mage_ai 内部依赖 | ✅ 能正确提取 import 行并打印路径 | 无偏差 |
| 读取所有依赖源码注入 System Prompt | ❌ `imported_file_paths` 恒为空，无依赖被读取注入 | **完全失效** |
| AI 基于完整上下文生成测试 | ❌ AI 仅能看到目标文件 + Few-shot 示范 | **认知缩水** |
| import → 文件路径映射正确 | ❌ 子包导入映射错误，即使修复 append 也会 `open()` 失败 | **部分失效** |

---

### 0.2 Windows CI 端到端配置差异核准（补齐完整表）

#### 完整三方配置对照表（逐字段核准，无遗漏）

对三份 Playwright 配置文件 + `build_and_test.yml` CI 工作流进行**逐字段交叉核准**，共 16 项配置维度：

| 配置维度 | 本地开发 `playwright.config.ts` | Linux CI `playwright.config.ci.ts` | Windows CI `playwright-windows.config.ci.ts` | 差异影响评估 |
|---------|-------------------------------|-----------------------------------|---------------------------------------------|-------------|
| **`expect.timeout`** | `45000`（45秒） | `45000`（45秒） | `45000`（45秒） | ✅ 全一致 |
| **`forbidOnly`** | `!!process.env.CI` | `!!process.env.CI` | `!!process.env.CI` | ✅ 全一致 |
| **`fullyParallel`** | `true` | `true` | `true` | ✅ 全一致 |
| **`projects`** | Chromium only | Chromium only | Chromium only | ✅ 全一致 |
| **`reporter`** | `'html'` | `'html'` | `'html'` | ✅ 全一致 |
| **`retries`** | `process.env.CI ? 2 : 0` | `process.env.CI ? 2 : 0` | `process.env.CI ? 2 : 0` | ✅ 全一致 |
| **`timeout`** | `100000`（100秒） | `100000`（100秒） | `100000`（100秒） | ✅ 全一致 |
| **`workers`** | `process.env.CI ? 1 : undefined` | `process.env.CI ? 1 : undefined` | `process.env.CI ? 1 : undefined` | ✅ 全一致 |
| **`use.trace`** | `'on'` | `'on'` | `'on'` | ✅ 全一致 |
| **`webServer.reuseExistingServer`** | `!process.env.CI` | `!process.env.CI` | `!process.env.CI` | ✅ 全一致 |
| --- | --- | --- | --- | --- |
| **`testDir`** 🔴 | `'./tests'`（4 spec） | `'./tests'`（4 spec） | `'./tests/basic'`（1 spec） | **Windows 测试集缩减 75%** |
| **`use.baseURL`** 🔴 | `'http://localhost:3000'` | `'http://localhost:6789'` | `'http://localhost:6789'` | 本地用 Next.js dev，CI 用生产服务 |
| **`webServer.command`** 🔴 | `'yarn run dev'` | `'python mage_ai/cli/main.py start test_project'` | `'cd venv3/Scripts && activate && cd ../../ && python mage_ai/cli/main.py start test_project'` | Windows 需显式激活 venv |
| **`webServer.cwd`** 🔴 | 未设置 | `'../../'` | `'../../'` | ✅ CI 间一致，本地不同 |
| **`webServer.url`** 🔴 | `'http://localhost:3000'` | `'http://localhost:6789'` | `'http://localhost:6789'` | 与 baseURL 对应 |
| **`webServer.env`** 🔴 | 未设置 env 对象 | `{ PYTHONPATH: '.', REQUIRE_USER_AUTHENTICATION: '1' }` | `{ INSTANCE_TYPE: 'web_server', PYTHONPATH: '.', REQUIRE_USER_AUTHENTICATION: '1' }` | **Windows 多设了 `INSTANCE_TYPE`** |
| --- | --- | --- | --- | --- |
| **Node 版本（CI YAML）** 🔴 | N/A | `18.18.0` | `20.15.1` | 前端构建工具链差异 |
| **Python 版本（CI YAML）** 🔴 | N/A | 矩阵 3.9-3.12，E2E 仅 3.10 | 仅 `3.10` | ✅ E2E Python 一致 |
| **后端单测（CI YAML）** 🔴 | N/A | ✅ `python -m unittest discover -s mage_ai` | ❌ **完全省略** | **Windows 后端零单测覆盖** |
| **集成单测（CI YAML）** 🔴 | N/A | ✅ `python -m unittest discover mage_integrations.tests` | ❌ **完全省略** | **Windows 集成零单测覆盖** |

#### 配置差异分类统计
| 分类 | 数量 | 具体维度 |
|-----|------|---------|
| ✅ 三平台完全一致 | 10 项 | expect.timeout、forbidOnly、fullyParallel、projects、reporter、retries、timeout、workers、use.trace、webServer.reuseExistingServer |
| 🔴 CI 间一致，与本地不同 | 4 项 | use.baseURL、webServer.command、webServer.url、webServer.cwd |
| 🔴 Linux CI vs Windows CI 差异 | 5 项 | testDir、webServer.env(INSTANCE_TYPE)、Node版本、后端单测、集成单测 |
| **合计** | **19 项** | |

#### 核准的 CI Job 结构证据

[build_and_test.yml#L111-L148](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/.github/workflows/build_and_test.yml#L111-L148) 中 `test_web_server_windows` Job：

```yaml
test_web_server_windows:
  runs-on: windows-latest
  strategy:
    matrix:
      python-version: ["3.10"]    # 仅 3.10，无矩阵
  steps:
    - uses: actions/checkout@v5
    - name: Set up Python 3.10
      uses: actions/setup-python@v6
      with:
        python-version: "3.10"
    - name: Set up Node
      uses: actions/setup-node@v6
      with:
        node-version: "20.15.1"   # Node 20，与 Linux 的 18.18.0 不同
    - name: Create Mage test project
      run: |
        Start-Process -Wait -FilePath "C:\Program Files\Git\unins000.exe" -ArgumentList "/SILENT"
        python -m venv venv3                          # 显式创建 venv
        venv3/Scripts/Activate.ps1                    # 手动激活
        $env:PYTHONPATH = "${env:PYTHONPATH};${pwd}"  # Windows 路径分号
        python -m pip install -r requirements.txt
        python mage_ai/cli/main.py init test_project
    - name: Build frontend, start server, and run Playwright tests
      run: |
        yarn install &&
        yarn export_prod &&
        yarn playwright install chromium &&
        yarn playwright test -c playwright-windows.config.ci.ts  # 专用配置
      working-directory: mage_ai/frontend/
    # ⚠️  完全没有：python -m unittest discover 步骤
    # ⚠️  完全没有：mage_integrations 测试步骤
```

**对比 Linux CI Job** [build_and_test.yml#L52-L109](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/.github/workflows/build_and_test.yml#L52-L109)：
- Linux 第 72-79 行：`Run unit tests` + `Run mage integrations unit tests` 两步完整执行
- Windows Job 中这两步**完全缺失**

---

### 0.3 配置差异对测试边界的具体影响

#### 影响一：Windows E2E 测试覆盖范围缩减 75%

Windows 的 `testDir: './tests/basic'` 与 Linux `testDir: './tests'` 导致：

| 测试文件 | Linux CI | Windows CI | 覆盖内容 |
|---------|----------|-----------|---------|
| `tests/pipelines.spec.ts` | ✅ 运行 | ❌ 跳过 | Overview 页面入口创建流水线 + Pipelines Dashboard 入口 |
| `tests/pipeline_runs.spec.ts` | ✅ 运行 | ❌ 跳过 | **example_pipeline 端到端运行流（核心业务路径）** |
| `tests/pages.spec.ts` | ✅ 运行 | ❌ 跳过 | 基础页面导航 |
| `tests/basic/pipelines.spec.ts` | ✅ 运行 | ✅ 运行 | 基础流水线 CRUD（单一入口） |

**业务影响**：Windows CI 完全无法检测 example_pipeline 运行成功与否，这是平台最核心的用户路径之一。

#### 影响二：Windows 平台后端代码**零单测覆盖**

| 检测维度 | Linux CI | Windows CI |
|---------|----------|-----------|
| 后端 Python 单测（mage_ai/） | ✅ 矩阵 ×4 版本全跑 | ❌ 0 个断言执行 |
| 集成单测（mage_integrations/） | ✅ Python 3.10 上跑 | ❌ 0 个断言执行 |
| 任何 Windows 特有路径（`os.name == 'nt'`、反斜杠路径、进程 `CREATE_NO_WINDOW` 等） | ❌ 无法覆盖 | ❌ 也无法覆盖（无单测） |

**典型盲区**：
- `mage_ai/shared/files.py` 中 Windows 特有的路径处理逻辑
- `mage_ai/services/spark/*` 中 Windows 下的进程启动参数
- `mage_ai/cli/main.py` 中 Windows 平台的事件循环选择

#### 影响三：环境变量不一致引发的测试有效性风险

```
INSTANCE_TYPE 差异：
  Linux CI:   未设置（默认值由代码 determine）
  Windows CI: 'web_server'
```

如果代码中存在 `os.getenv('INSTANCE_TYPE') == 'web_server'` 的条件分支（如 scheduler 角色判断、并发限制、资源配额计算等），两个平台的测试就运行在**不同代码分支**上，失去可比性。

#### 影响四：E2E 功能开关的固定盲区（两平台共有）

[tests/utils.ts#L15-L23](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/utils.ts#L15-L23) 中的 `ignoreKeys` 定义了**永远不被 E2E 测试启用的功能**：

```typescript
const featuresFiltered = ignoreKeys(FeatureUUIDEnum, [
  'CODE_BLOCK_V2',                    // 新版代码块编辑器
  'COMMAND_CENTER',                   // 命令中心 UI
  'COMPUTE_MANAGEMENT',               // 计算资源管理
  'CUSTOM_DESIGN',                    // 自定义设计系统
  'DBT_V2',                           // DBT v2 集成
  'GLOBAL_HOOKS',                     // 全局钩子
  'NOTEBOOK_BLOCK_OUTPUT_SPLIT_VIEW', // Notebook 分栏视图
]);
```

**意味着**：即使这 7 个功能在代码中存在缺陷，E2E 测试也**永远无法检测**到——这是硬编码的覆盖盲区，与平台无关。

---

## 一、前端端到端测试（Playwright E2E）

### 1.1 框架与配置体系

Mage AI 前端采用 **Playwright** 作为端到端测试框架，存在三套配置文件：

| 配置文件 | 用途 | Base URL | Web Server 命令 | Workers |
|---------|------|----------|----------------|---------|
| [playwright.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/playwright.config.ts) | 本地开发调试 | `http://localhost:3000` | `yarn run dev`（Next.js dev server） | 本地并行 |
| [playwright.config.ci.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/playwright.config.ci.ts) | CI 环境 | `http://localhost:6789` | `python mage_ai/cli/main.py start test_project`（生产模式） | 串行（workers=1） |
| [playwright-windows.config.ci.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/playwright-windows.config.ci.ts) | Windows CI | `http://localhost:6789` | `cd venv3/Scripts && activate && cd ../../ && python mage_ai/cli/main.py start test_project`（需显式激活 venv） | 串行（workers=1） |

> **⚠️ 核准提示**：完整的 19 项配置维度三方对比表（含 CI YAML 层面差异）请参考 [第零章 0.2 节](#02-windows-ci-端到端配置差异核准补齐完整表)。

**CI 核心配置特征（Linux CI）**：

```typescript
// playwright.config.ci.ts
{
  expect: { timeout: 45000 },      // 断言超时 45 秒
  timeout: 100000,                 // 单测试超时 100 秒
  retries: process.env.CI ? 2 : 0, // CI 失败自动重试 2 次
  forbidOnly: !!process.env.CI,    // 禁止 CI 中使用 test.only
  fullyParallel: true,             // 文件级并行
  workers: process.env.CI ? 1 : undefined, // CI 串行避免资源竞争
  use: {
    baseURL: 'http://localhost:6789',
    trace: 'on',                   // 始终录制追踪用于调试
  },
  webServer: {
    command: 'python mage_ai/cli/main.py start test_project',
    env: {
      PYTHONPATH: '.',
      REQUIRE_USER_AUTHENTICATION: '1', // CI 强制启用用户认证
    },
    url: 'http://localhost:6789',
    reuseExistingServer: !process.env.CI,
  }
}
```

**关键设计意图**：
- **CI 使用生产模式启动服务**（6789端口），而非 Next.js dev server，模拟真实部署环境
- **强制认证**：`REQUIRE_USER_AUTHENTICATION=1`，测试带权限的真实用户流程
- **Trace 全开**：失败时可通过 Playwright Trace Viewer 复现完整操作
- **长超时**：E2E 涉及服务启动、页面渲染、异步加载，超时阈值高

**Windows CI 特有差异（与第零章核准结论统一）**：
- 🔴 **测试集缩减**：`testDir: './tests/basic'`，仅运行 `tests/basic/pipelines.spec.ts`，不跑 `pipeline_runs.spec.ts` 等核心流程
- 🔴 **环境变量不一致**：额外设置 `INSTANCE_TYPE: 'web_server'`，可能导致与 Linux CI 运行在不同代码分支
- 🔴 **Node 版本差异**：Node 20.15.1 对比 Linux 的 Node 18.18.0
- 🔴 **需显式激活 venv**：启动命令前置 `cd venv3/Scripts && activate && cd ../../`
- 🔴 **后端单测全跳**：CI YAML 中完全省略 `python -m unittest discover` 步骤

### 1.2 测试 Fixture 体系

[tests/base.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/base.ts) 定义了全局测试 Fixture：

```typescript
export const test = base.extend<{
  failOnClientError: boolean;
  settingFeaturesToDisable: TSettingFeaturesToDisable,
}>({
  failOnClientError: true,   // 默认：页面 JS 报错即测试失败
  settingFeaturesToDisable: {
    [FeatureUUIDEnum.LOCAL_TIMEZONE]: true, // 禁用本地时区，保证时间断言一致性
  },
  page: async ({ page, failOnClientError, settingFeaturesToDisable }, use) => {
    // 1. 捕获所有页面 JS 错误
    const pageErrors: Error[] = [];
    page.addListener('pageerror', (error) => {
      pageErrors.push(error);
    });

    // 2. 统一登录流程：admin@admin.com / admin
    await page.goto('/sign-in');
    await page.getByPlaceholder('Email').click();
    await page.getByRole('textbox').first().fill('admin@admin.com');
    await page.getByRole('textbox').first().press('Tab');
    await page.locator('input[type="password"]').fill('admin');
    await page.locator('input[type="password"]').press('Enter');

    // 3. 等待登录完成（通过"New"按钮可见性判断）
    await expect(page.getByRole('button', { name: 'New' })).toBeVisible();

    // 4. 禁用指定功能开关，消除测试不确定性
    await enableSettings(page, settingFeaturesToDisable);

    await use(page);  // 注入到测试用例

    // 5. 测试后断言：无客户端 JS 错误
    if (failOnClientError) {
      expect(pageErrors).toHaveLength(0);
    }
  },
});
```

**边界设计亮点**：

1. **零 JS 错误约束**：`failOnClientError=true` 将前端控制台错误视为测试失败，比功能断言更严格
2. **默认禁用本地时区**：消除时区差异导致的时间相关断言失败
3. **统一登录 Fixture**：所有 E2E 测试共享同一管理员账号，避免重复登录代码

### 1.3 测试用例结构与断言模式

**现有测试套件**：

| 测试文件 | 覆盖场景 | 核心断言 |
|---------|---------|---------|
| [pipelines.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/pipelines.spec.ts) | 从 Overview/Pipelines 页面创建删除流水线 | URL 路由断言、文本可见性、对话框处理 |
| [pages.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/pages.spec.ts) | 基础页面导航 | - |
| [pipeline_runs.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/pipeline_runs.spec.ts) | example_pipeline 运行完整生命周期 | Trigger 状态流转：N/A → running → completed |
| [basic/pipelines.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/basic/pipelines.spec.ts) | 基础流水线 CRUD | 同上 |

**断言模式分析**（以 pipelines.spec.ts 为例）：

```typescript
// 1. 路由断言：通过 URL 路径结构判断导航正确性
const pathStr = await page.evaluate(() => document.location.pathname);
const path = pathStr.split('/');
expect(path[1]).toBe('pipelines');
expect(path[3]).toBe('edit');

// 2. 内容断言：验证页面包含流水线名称
await expect(page.locator('[id="__next"]')).toContainText(path[2]);

// 3. 可见性断言：元素状态变化检测
await expect(page.getByText('Name', { exact: true })).toBeVisible();
await expect(page.getByRole('cell', { name: pipelineName })).toBeVisible();
await expect(page.getByRole('cell', { name: pipelineName })).toBeHidden();

// 4. 对话框断言：精确匹配确认框文本
page.on('dialog', async dialog => {
  if (dialog.message() === `Are you sure you want to delete pipeline ${pipelineName}?`) {
    await dialog.accept();
  } else {
    throw new Error(`Unexpected dialog: ${dialog.message()}`); // 非预期对话框直接失败
  }
});

// 5. 状态流断言（pipeline_runs.spec.ts）：验证状态机流转
await expect(page.locator('#pipeline-triggers-row-0')).toContainText('running');
await expect(page.getByRole('button', { name: 'Running' })).toBeVisible();
await expect(page.getByRole('button', { name: 'Done' })).toBeVisible();
await expect(page.locator('#pipeline-triggers-row-0')).toContainText('completed');
```

**E2E 边界约束**：

| 约束维度 | 具体限制 | 设计意图 |
|---------|---------|---------|
| 浏览器 | 仅 Chromium，不测 Firefox/WebKit | 降低 CI 资源消耗和 flaky 率 |
| 并行度 | CI 单 Worker，不并行测试用例 | 避免共享后端状态冲突 |
| 重试 | CI 失败自动重试 2 次 | 容忍网络/时序类偶发失败 |
| 账号 | 硬编码 admin@admin.com/admin | 与后端 init 项目的默认用户匹配 |
| 追踪 | `trace: 'on'` 全量录制 | 失败后可完整回放复现 |

---

## 二、自动生成单测脚本（AI 辅助测试生成）

### 2.1 脚本架构总览

[scripts/server/write_tests.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py) 是一个基于 **Claude AI (claude-3-opus-20240229)** 的单元测试自动生成工具。

> **⚠️ 重要核准结论（与第零章统一）**：设计了依赖源码注入能力，但**因代码 BUG 实际未生效**。详见 [第零章 0.1 节](#01-自动生成脚本依赖上下文注入实际完全失效统一结论)。

**设计预期的执行流程**（代码逻辑设计）：

```
输入：目标 Python 文件路径
    ↓
1. 读取源代码内容
    ↓
2. extract_mage_ai_imports()：解析 mage_ai 内部依赖导入
    ↓
3. build_file_path_from_import()：将每个 import 转为源码文件路径
    ↓
4. build_system_prompt()：读取所有依赖源码，作为上下文注入 System Prompt
    ↓
5. build_prompt()：构造 Few-shot Prompt（示例代码 + 示例测试）
    ↓
6. build_test()：调用 Claude API 生成测试代码
    ↓
7. 解析 AI 返回：提取 <test>...</test> 标签内代码
    ↓
8. 写入 mage_ai/tests/ 对应目录的 test_*.py 文件
    ↓
9. run_shell_command()：自动执行 unittest 验证生成结果
```

**实际执行的流程**（代码 BUG 导致）：

```
输入：目标 Python 文件路径
    ↓
1. 读取源代码内容
    ↓
2. extract_mage_ai_imports()：解析 mage_ai 内部依赖导入（仅打印，未存入列表）
    ↓
3. build_file_path_from_import()：将每个 import 转为源码文件路径（仅打印，未使用）
    ↓
4. build_system_prompt([])：传入空列表，<documents> 标签内容为空
    ↓
5. build_prompt()：构造 Few-shot Prompt（仅包含目标文件 + 示范对）
    ↓
6. build_test()：调用 Claude API 生成测试代码（零依赖上下文）
    ↓
7. 解析 AI 返回：提取 <test>...</test> 标签内代码
    ↓
8. 写入 mage_ai/tests/ 对应目录的 test_*.py 文件
    ↓
9. run_shell_command()：自动执行 unittest 验证生成结果
```

### 2.2 Prompt 工程设计

**Few-shot 学习策略**：脚本内置了一对「代码-测试」示例作为示范：

```python
# 示范输入：shared/retry.py 的 retry 装饰器
# 示范输出：shared/test_retry.py 的 RetryTests

<test>
from unittest.mock import call, patch

from mage_ai.shared.retry import retry
from mage_ai.tests.base_test import TestCase


class RetryTests(TestCase):
    @patch('time.sleep')
    def test_retry(self, mock_sleep):
        @retry(retries=2, max_delay=40)
        def test_func():
            raise Exception('error')
            return

        with self.assertRaises(Exception):
            test_func()
            mock_sleep.assert_has_calls([call(5), call(10), call(20)])
</test>
```

**Prompt 构造的三大原则**（见 build_prompt 注释）：

1. **问题后置**：将任务要求放在 Prompt 末尾，提升 AI 注意力
2. **强制引用先行**：要求 AI 先找到相关代码引用再回答，降低幻觉
3. **角色预热**："Instruct Claude to read the document carefully, as it will be asked questions later"

**System Prompt 上下文注入（与第零章核准结论统一）**：

| 设计预期 | 实际代码行为 |
|---------|-------------|
| `build_system_prompt()` 将目标文件及其所有 mage_ai 内部依赖的源代码完整注入 | ❌ 因 `imported_file_paths` 恒为空，`<documents>` 标签内容为空，零依赖注入 |
| AI 理解真实实现细节，生成更准确的测试 | ❌ AI 仅基于目标文件本身 + Few-shot 示范生成 |

### 2.3 文件路径映射规则

```python
def build_file_path_from_import(import_line, base_path=''):
    # from mage_ai.shared.retry import retry
    # → 'mage_ai.shared.retry'
    # → 'mage_ai/shared/retry.py'
    normalized_line = import_line.strip().replace('import ', '').replace('from ', '').split(' ')[0]
    path = normalized_line.replace('.', '/') + '.py'
    return os.path.join(base_path, path)

# 输出路径规则：
# 输入: mage_ai/shared/utils.py
# 输出: mage_ai/tests/shared/test_utils.py
```

**核准的映射边界缺陷（与第零章统一）**：
- ✅ 对 `import mage_ai.shared.retry`（模块文件）：正确映射到 `mage_ai/shared/retry.py`
- ❌ 对 `from mage_ai.settings import X`（子包导入）：错误映射到 `mage_ai/settings.py`（实际应为 `mage_ai/settings/__init__.py`）
- ❌ 对 `from mage_ai.orchestration.db import safe_db_query`（多层子包）：错误映射到 `mage_ai/orchestration/db.py`

### 2.4 自动化验证闭环

生成完成后，脚本会**自动运行 unittest** 验证：

```python
command = f'./scripts/server/py.sh -m unittest {test_file_path}'
output = run_shell_command(command, raise_on_error=False)
```

**边界约束（与第零章核准结论统一）**：
- **模型限制**：`max_tokens=4096`，单个文件生成的测试代码约 1000 行以内
- **依赖注入实际状态**：⚠️  零依赖注入（BUG 导致），第三方库和内部依赖均未注入
- **路径映射实际状态**：⚠️  子包导入映射错误（即使修复 append BUG，也会 `FileNotFoundError`）
- **手动目标文件**：脚本主函数使用硬编码文件列表 `mage_ai/data_preparation/models/utils.py` 等，注释中提供了 `git diff` 过滤新增文件的自动化方案

---

## 三、数据库边界约束深度分析

### 3.1 数据库连接分层决策树

[orchestration/db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/__init__.py#L22-L52) 中的连接优先级：

```
┌─ is_test_mage() == True?
│   ├── YES → sqlite:///test.db（单元测试专用，优先级最高）
│   └── NO ─┐
│           ├─ DATABASE_CONNECTION_URL 环境变量存在? → 使用该 URL
│           ├─ PostgreSQL 可用?（AWS/Azure/本地 env）→ PostgreSQL
│           └─ SQLite fallback:
│              ├─ 开发目录存在? → sqlite:///mage_ai/orchestration/db/mage-ai.db
│              ├─ 根目录 mage-ai.db 存在? → sqlite:///mage-ai.db
│              └─ 新项目 → sqlite:///{variables_dir}/mage-ai.db
└─ 所有 PostgreSQL 连接追加：pool_size=50 + timezone=utc
```

### 3.2 查询重试与回滚机制

`safe_db_query` 装饰器处理三大异常类型：

[db/__init__.py#L155-L190](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/__init__.py#L155-L190)

```python
DB_RETRY_COUNT = 2

def safe_db_query(func):
    def func_with_rollback(*args, **kwargs):
        retry_count = 0
        while True:
            try:
                return func(*args, **kwargs)
            except (
                sqlalchemy.exc.OperationalError,    # 连接丢失、超时
                sqlalchemy.exc.PendingRollbackError, # 前置事务失败需回滚
                sqlalchemy.exc.InternalError,        # 死锁、内部状态异常
            ) as e:
                db_connection.session.rollback()      # 关键：先回滚再重试
                if retry_count >= DB_RETRY_COUNT:     # 最多重试 2 次（共 3 次尝试）
                    raise e
                retry_count += 1
    return func_with_rollback
```

**异步版本**：`safe_db_query_async` 行为一致，包装协程函数。

**全项目使用分布**（grep 统计）：

| 模块文件 | `@safe_db_query` 装饰器数量 |
|---------|---------------------------|
| models/schedules.py | 21 处 |
| models/oauth.py | 若干 |
| models/secrets.py | 1 处 |
| models/utils.py | 3 处 |
| models/schedules_project_platform.py | 1 处 |

**典型应用场景**：PipelineRun 状态更新、OAuth 令牌刷新、配置写入等涉及并发写入的操作。

### 3.3 ORM 模型级事务边界

[models/base.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/models/base.py#L51-L103) 中 BaseModel 的每个持久化方法均有 try-commit/except-rollback：

```python
def save(self, commit=True) -> None:
    # 先执行字段级 validate_* 校验
    for table_column in self.__table__.columns:
        column = table_column.name
        value = getattr(self, column)
        if value is None and hasattr(self, f'validate_{column}'):
            getattr(self, f'validate_{column}')(column, value)

    self.session.add(self)
    if commit:
        try:
            self.session.commit()
        except Exception as e:
            self.session.rollback()  # 单操作失败不污染全局 session
            raise e

# update / delete 采用完全相同的模式
```

### 3.4 数据库迁移锁

[database_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/database_manager.py#L31-L73) 通过**咨询锁（Advisory Lock）** 防止多实例并发迁移：

```python
@contextlib.contextmanager
def create_db_lock(session, lock: DBLocks, lock_timeout: int = 30000):
    conn = session.get_bind().connect()
    dialect = conn.dialect
    try:
        if dialect.name == 'postgresql':
            conn.execute(text('SET LOCK_TIMEOUT to :timeout'), {'timeout': lock_timeout})
            conn.execute(text('SELECT pg_advisory_lock(:id)'), {'id': lock.value})
        elif dialect.name == 'mysql' and dialect.server_version_info >= (5, 6):
            conn.execute(text('SELECT GET_LOCK(:id, :timeout)'), {'id': str(lock), 'timeout': lock_timeout})
        lock_acquired = True
        yield
    finally:
        # 对应解锁逻辑
        # Postgres: pg_advisory_unlock + 解锁失败抛异常
        # MySQL: RELEASE_LOCK
```

### 3.5 查询缓存层

[cache.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/cache.py) 在单元测试中可控制的内存缓存：

```python
class CachingQuery(Query):
    def all(self, *args, **kwargs):
        if self.cache:
            # cache_key = hash(SQL语句 + JSON序列化参数)
            compiler = self.statement.compile()
            cache_key_1 = compiler.string
            cache_key_2 = simplejson.dumps(compiler.params or {}, default=encode_complex, ignore_nan=True)
            self.cache_key = str(hash(f'{cache_key_1}:{cache_key_2}'))

            results = self.session.get_results_from_cache(self.cache_key)
            if results:  # CACHE HIT
                return results

        results = super().all(*args, **kwargs)
        if self.cache and self.cache_key:
            self.session.cache_results(self.cache_key, results)  # CACHE MISS 写入
        return results
```

**测试中的缓存控制接口**（通过 db_connection 调用）：
- `db_connection.start_cache()`：启用内存缓存字典
- `db_connection.stop_cache()`：清空缓存
- **注意**：测试基类默认不启动缓存，需显式调用

### 3.6 测试数据库清理生命周期

| 阶段 | 操作 | 代码位置 |
|-----|------|---------|
| setUpClass | `database_manager.run_migrations(log_level=logging.ERROR)`：创建表结构 | [base_test.py#L75](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/base_test.py#L75-L75) |
| setUpClass | `db_connection.start_session(force=True)`：强制新建 session | [base_test.py#L76](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/base_test.py#L76-L76) |
| tearDownClass | `db_connection.close_session()` | [base_test.py#L85](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/base_test.py#L85-L85) |
| tearDownClass | `Path(TEST_DB).unlink()`：物理删除 test.db 文件 | [base_test.py#L87-L88](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/base_test.py#L87-L88) |

**关键约束**：
- **类级别隔离**：每个测试类独享完整 DB 生命周期，测试方法之间共享数据（非事务回滚策略）
- **无测试间事务回滚**：依赖彻底删除 + 重建 DB 文件，而非每个测试 ROLLBACK
- **SQLite 多线程模式**：`check_same_thread=False`，支持异步测试中的跨线程 DB 访问

---

## 四、外部依赖隔离策略

### 4.1 Mock 模式分层体系

项目中使用 `unittest.mock` 的四种模式，按使用频率排序：

#### 模式一：`with patch.object()` — 方法级局部替换（最常用）

**典型场景**：所有 streaming Source/Sink 的连接初始化 Mock

[tests/streaming/sources/test_rabbitmq.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/streaming/sources/test_rabbitmq.py) 模式：

```python
# 模式：所有外部连接统一在 init_client 方法中，Mock 该方法即可
with patch.object(RabbitMQSource, 'init_client') as mock_init_client:
    source = RabbitbitMQSource(config=...)
    # 断言构造函数调用了 init_client
    mock_init_client.assert_called_once()

    # 测试错误场景：init_client 抛异常
with patch.object(RabbitMQSource, 'init_client', side_effect=Exception('Connection refused')):
    with self.assertRaises(Exception) as context:
        source = RabbitbitMQSource(config=...)
    self.assertTrue('Connection refused' in str(context.exception))
```

**全 streaming 模块一致性**：14 种 Source + 14 种 Sink 全部采用 `init_client()` 作为唯一外部连接入口，Mock 模式完全统一。

#### 模式二：`@patch('module.path.ClassName')` — 装饰器级全局替换

[tests/streaming/sources/test_kafka_raw_value.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/streaming/sources/test_kafka_raw_value.py#L10)：

```python
class KafkaSourceTests(TestCase):
    @patch('mage_ai.streaming.sources.kafka.KafkaConsumer')
    def test_kafka_raw_value_extraction(self, mock_consumer_class):
        mock_consumer = MagicMock()
        mock_message = MagicMock()
        mock_message.value = b'raw_bytes_payload'
        mock_consumer.__iter__.return_value = iter([mock_message])
        mock_consumer_class.return_value = mock_consumer

        source = KafkaSource(config=...)
        # 验证提取逻辑正确处理原始字节
```

#### 模式三：`MagicMock` + 属性预设 — 复杂对象模拟

[tests/orchestration/test_execution_process_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/orchestration/test_execution_process_manager.py#L57-L70)：

```python
proc_block_terminated = MagicMock()
proc_block_terminated.is_alive = MagicMock(return_value=False)  # 已死亡
proc_block_cancelled = MagicMock()
proc_block_live = MagicMock()                                 # 存活

proc_pipeline_terminated = MagicMock()
proc_pipeline_terminated.is_alive = MagicMock(return_value=False)
proc_pipeline_cancelled = MagicMock()
proc_pipeline_live = MagicMock()

manager.set_block_process(pipeline_run1.id, block_runs[0].id, proc_block_terminated)
# ... 设置所有进程后执行清理
manager.clean_up_processes(include_child_processes=False)

# 断言清理结果符合预期
self.assertEqual(manager.block_processes, {pipeline_run1.id: {block_runs[2].id: proc_block_live}})
```

#### 模式四：`@patch.dict(os.environ, {...})` — 环境变量模拟

[tests/services/k8s/test_job_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/services/k8s/test_job_manager.py#L114-L150)：

```python
@patch.dict(os.environ, {'HOSTNAME': 'mage-server'})
class JobManagerTests(TestCase):
    @patch('mage_ai.services.k8s.job_manager.config')
    @patch('mage_ai.services.k8s.job_manager.client')
    def test_init(self, mock_config, mock_client):
        # 嵌套多层 patch，参数顺序从下往上
        mock_batch_api_client = MagicMock()
        with patch.object(mock_client, 'BatchV1Api', return_value=mock_batch_api_client):
            with patch.object(mock_client, 'CoreV1Api', return_value=mock_core_v1_api_client):
                # ... 断言
```

### 4.2 调用验证断言模式

Mock 对象的行为验证断言（与数据断言分离）：

| 断言方法 | 用途 | 示例 |
|---------|------|------|
| `assert_called_once()` | 恰好调用 1 次 | `mock_load_config.assert_called_once()` |
| `assert_called_once_with(...)` | 以指定参数调用 1 次 | `mock_read_namespaced_pod_method.assert_called_once_with('mage-server', 'test_namespace')` |
| `assert_has_calls([call(...), ...])` | 按顺序多次调用 | `mock_sleep.assert_has_calls([call(5), call(10), call(20)])` |
| `assert_not_called()` | 完全未调用 | `mock_sync.assert_not_called()` |
| `.call_count` | 调用次数数值断言 | `self.assertEqual(mock_load_data.call_count, 2)` |

### 4.3 单元测试 vs 集成测试的边界

#### 边界划分规则

| 维度 | 单元测试（mage_ai/tests） | 集成测试（mage_integrations/tests） |
|-----|-------------------------|-----------------------------------|
| 数据库 | SQLite `test.db`，文件级别删除重建 | 通常需要真实目标 DB（PostgreSQL/MySQL等） |
| 外部服务 | **100% Mock**（init_client/网络请求/K8s API） | 实际连接（需要凭据，仅本地/专项 CI 运行） |
| 测试基类 | `TestCase/DBTestCase/AsyncDBTestCase` | `BaseSourceTests`（标准 unittest.TestCase） |
| 数据样本 | Faker 生成 + 内存对象 | 真实 JSON fixtures（samples/ 目录） |
| 运行频率 | 每次提交的 PR 必跑 | 按需/夜间流水线 |

#### mage_integrations 单元测试的 Mock 模式

[mage_integrations/tests/sources/test_base.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_integrations/mage_integrations/tests/sources/test_base.py#L110-L141)：

```python
def test_process_execute_test_connection(self):
    source = Source(test_connection=True)
    # 用 5 层嵌套 patch 屏蔽完整调用链
    with patch.object(source, 'test_connection') as mock_test_connection:
        with patch.object(source, 'discover_streams') as mock_discover_streams:
            with patch.object(source, 'discover') as mock_discover:
                with patch.object(source, 'load_data') as mock_load_data:
                    with patch.object(source, 'sync') as mock_sync:
                        source.process()
                        # 断言正确分支调用 + 其他分支零调用
                        mock_test_connection.assert_called_once()
                        mock_discover_streams.assert_not_called()
                        mock_discover.assert_not_called()
                        mock_load_data.assert_not_called()
                        mock_sync.assert_not_called()
```

### 4.4 环境边界的 Patch 激活模式

[ProjectPlatformMixin](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/shared/mixins.py#L349-L392) 使用**类装饰器 Patch** 切换全局功能开关：

```python
@patch('mage_ai.settings.platform.project_platform_activated', lambda: True)
@patch('mage_ai.settings.repo.project_platform_activated', lambda: True)
@patch('mage_ai.orchestration.db.models.schedules.project_platform_activated', lambda: True)
class ProjectPlatformMixin(AsyncDBTestCase):
    """
    三个关键模块同时 patch，确保多项目平台模式的所有代码分支都被激活。
    这是测试"功能开关"类边界条件的标准模式。
    """
    @classmethod
    def setUpClass(self):
        super().setUpClass()
        self.initialize_settings()  # 写入临时 platform_settings.yaml
```

### 4.5 外部依赖隔离的黄金规则（从代码归纳）

1. **统一入口原则**：所有外部连接初始化收敛到单一方法（`init_client()`），Mock 一点即全局隔离
2. **分层 Mock 原则**：环境变量 → 模块导入 → 类方法 → 对象属性，按需选择层级
3. **负向断言原则**：验证**不该调用**的方法确实未调用（`assert_not_called`）
4. **时序断言原则**：对重试/状态机类代码使用 `assert_has_calls` 验证调用序列
5. **装饰器优先原则**：类级开关用 `@patch` 装饰器，方法级替换用 `with patch.object()` 上下文

---

## 五、关键代码索引

| 分析领域 | 文件路径 | 核心行号范围 |
|---------|---------|-------------|
| E2E 配置（CI） | [playwright.config.ci.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/playwright.config.ci.ts) | 1-58 |
| E2E 测试 Fixture | [tests/base.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/base.ts) | 1-44 |
| E2E 流水线测试 | [pipelines.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/pipelines.spec.ts) | 1-46 |
| E2E 流水线运行测试 | [pipeline_runs.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/frontend/tests/pipeline_runs.spec.ts) | 1-23 |
| AI 生成单测脚本 | [write_tests.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/scripts/server/write_tests.py) | 1-313 |
| DB 连接决策 | [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/__init__.py) | 22-52 |
| DB 重试装饰器 | [db/__init__.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/__init__.py) | 155-190 |
| DB 迁移锁 | [database_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/database_manager.py) | 31-115 |
| DB 查询缓存 | [cache.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/cache.py) | 1-77 |
| ORM 事务边界 | [models/base.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/orchestration/db/models/base.py) | 51-103 |
| 测试基类 DB 生命周期 | [base_test.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/base_test.py) | 60-90 |
| K8s Mock 模式范例 | [test_job_manager.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/services/k8s/test_job_manager.py) | 1-150 |
| Integration 测试基类 | [mage_integrations tests/test_base.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_integrations/mage_integrations/tests/sources/test_base.py) | 53-150 |
| 平台模式 Mixin Patch | [mixins.py](file:///d:/fz/0601/solo-dogfeeding/code/332-mage-ai/mage_ai/tests/shared/mixins.py) | 349-423 |

# MindQuantum 网站

MindQuantum 网站和双语文档的 Astro + Jupyter Book 单一代码库。

## 概述

- Astro 驱动主页和整个网站外壳。
- Jupyter Book 仅构建双语教程（英文+中文）。
- Jupyter Book 还会构建位于 `/courses` 下的独立课程笔记本。
- Sphinx 使用内部 `mqdocs` 扩展构建两个项目（英文+中文）的 API 参考。
- 共享设计令牌保持两者之间的视觉一致性。
- GitHub Pages 工作流同时构建和部署这两个输出。

## 本地开发

先决条件：Node 18+ 和 Python 3.9+。

1. 安装 Node 依赖

```bash
npm install
```

2. (可选) 创建 Python 虚拟环境并安装 Jupyter Book

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyter-book==1.0.4 sphinx-copybutton sphinx-design sphinx-thebe mindspore mindquantum
```

3. 同步文档内容（开发期间可选）

该仓库在首次运行时可以自动将上游源克隆到本地缓存中。要同步教程和 API 文档：

```bash
npm run sync:all            # 如果缺失则克隆，如果存在则重用缓存
# 或在同步前更新到最新的上游
python scripts/sync_all.py --update
```

4. 构建文档（教程 + API）

```bash
npm run build:docs   # 自动同步上游，构建 JB（教程）+ Sphinx（API）
```

5. 运行网站

```bash
npm run dev
```

## 内容同步

- 自动克隆：上游源使用 `scripts/upstreams.json` 缓存到 `.upstreams/` 下。
- 教程：`scripts/sync_mindquantum_from_msdocs.py` 从缓存的 `mindspore-docs` 克隆中获取 Sphinx 源。
- API：`scripts/sync_mindquantum_api.py` 从缓存的 `mindquantum` 克隆中获取 API 源。它还会同步到 `docs/api-en/api_python_en` 和 `docs/api-zh/api_python` 的独立 Sphinx 项目中。

### API (Sphinx) 项目

除了 Jupyter Books，API 参考还作为两个 Sphinx 项目构建，使用一个干净的内部扩展 (`mqdocs`)：

- `docs/api-en/` – 英文 API，使用从文档字符串派生第三列的 `ms*autosummary` 指令
- `docs/api-zh/` – 中文 API，使用从本地 RST 页面读取摘要的 `mscnautosummary` 指令

共享资产位于 `docs/_ext`、`docs/_templates` 和 `docs/_static` 下。

本地构建示例：

```bash
# 确保源已同步到 docs/api-*/
python scripts/sync_mindquantum_api.py

# 构建英文 API（集中在 docs/_build 下）
sphinx-build -b html docs/api-en docs/_build/api/en

# 构建中文 API（集中在 docs/_build 下）
sphinx-build -b html docs/api-zh docs/_build/api/zh
```

该扩展避免了猴子补丁，并实现了稳定的 `mscnautosummary`/`ms*autosummary` 指令，以及仅用于这些指令的最小存根生成器。标准 autosummary 保持不变。

## 构建和部署

- `npm run build:all` 会将 Jupyter Books（教程 + 课程）和 Sphinx 构建到 `public/{docs,courses}/**`，然后将 Astro 构建到 `dist/`。临时工件集中在 `docs/_build/` 与 `courses/_build/` 下。
- GitHub Actions 工作流 `.github/workflows/deploy.yml` 会构建两者并将 `dist/` 文件夹部署到 GitHub Pages。它会通过检测 `public/CNAME` 中提交的自定义域名，在构建前导出 `ASTRO_BASE` 与 `SITE_URL`。

## 文档路由

- 教程 (Jupyter Book)：`/docs/en` 和 `/docs/zh` (来自 `public/docs/en` 和 `public/docs/zh`)。
- 课程 (Jupyter Book)：`/courses` (来自 `public/courses/zh`)。
- API (Sphinx)：`/docs/api/en` 和 `/docs/api/zh` (来自 `public/docs/api/en` 和 `public/docs/api/zh`)。
- 网站头部应链接到两种语言的教程和 API。

## 主题

- 共享 CSS 令牌位于 `src/styles/tokens.css`。
- 一个小的构建步骤会将令牌复制到 `docs/_static/mq-variables.css`，以便 Jupyter Book 可以使用它们。
- Jupyter Book 加载 `mq-variables.css` 和 `mq-theme.css` 以使页面样式与主页保持一致。

## 结构

- `src/` – Astro 页面、布局和样式。
- `public/` – 静态资产。构建的文档会复制到 `public/docs`（被 Git 忽略）。
- `docs/` – 文档工作区：
  - Jupyter Book 项目：`docs/en` 和 `docs/zh`（仅限教程）
  - Sphinx API 项目：`docs/api-en` 和 `docs/api-zh`
  - 共享资产：`docs/_ext`、`docs/_static`、`docs/_templates`
- `scripts/` – 辅助脚本（令牌同步、上游同步、本地构建）。

## 交互式教程 (Thebe + Binder)

教程页面现在已启用 Thebe，用户可以直接在页面上点击代码单元格的“运行”按钮。这使用 Binder 为每个用户会话启动一个实时 Jupyter 内核。

- 切换：每个教程页面上都会出现一个“启动/激活”按钮来初始化 Thebe。激活后，每个代码单元格都会显示一个“运行”按钮。
- 范围：执行由客户端发起；笔记本不会自动执行。这使得构建具有确定性，并能更好地应对流量。
- 环境：Binder 环境由仓库 https://github.com/MindQuantum-HiQ/mq-env 进行配置和控制。

### 增强的用户体验附加功能

- 每个单元格的运行按钮：在 `docs/_static/mq-thebe.js` 中实现，并由 `docs/_static/mq-thebe.css` 样式化。它在 Python 代码单元格和笔记本单元格上叠加一个小的“运行”按钮。首次点击时，它会自动激活 Thebe，然后运行被点击的单元格。

- 页面横幅：教程页面顶部有一个紧凑的横幅，宣传该页面可在浏览器中运行。它提供“激活”和“运行所有示例”操作，并镜像内置的 Thebe 状态行。
- 包含：这两个资产都在 `docs/en/_config.yml` 和 `docs/zh/_config.yml` 中注册，位于 `sphinx.config.html_css_files` 和 `sphinx.config.html_js_files` 下。
- 可访问性：按钮包含 `aria-label`；颜色使用共享令牌以确保对比度。默认情况下，叠加层会避开复制按钮，并且如果给定页面上没有 Thebe，则会优雅地回退。

自定义和维护：

- 要禁用横幅：从两个 Jupyter Book 配置中的 `html_css_files`/`html_js_files` 中移除 `mq-thebe.css`/`mq-thebe.js`。
- 要限制每个单元格的按钮：调整 `mq-thebe.js` 中（函数 `attachRunButtons`）的选择器逻辑，使其仅匹配您偏好的模式（例如，仅 `.cell` 或仅 `code.language-python`）。`.thebe-ignored` 内部的单元格会自动跳过。
- 该脚本避免了私有的 Thebe 内部机制：它触发标准的启动按钮并点击单元格内置的 `.thebe-run-button`。这使其在 Thebe/Jupyter Book 更新时仍能保持弹性。

## 量子图形化编程（主页）

主页提供交互式电路搭建器，可在浏览器中完成量子图形化编程。

- 支持拖拽量子门、选择控制/目标，并即时查看仿真结果。
- 以 Web 组件 `mq-circuit-builder` 实现：
  - 核心逻辑：`src/components/circuit/QuantumCircuitElement.ts`
  - 样式：`src/components/circuit/styles.css`
- 通过 `src/components/QuantumCircuitBuilder.astro` 封装，并在 `src/pages/index.astro` 与 `src/pages/[lang]/index.astro` 渲染。
- 文案与国际化来自 `src/locales/home.ts` 的 `builder` 段。
- 完全在浏览器端运行，无需服务器或 Python 环境。

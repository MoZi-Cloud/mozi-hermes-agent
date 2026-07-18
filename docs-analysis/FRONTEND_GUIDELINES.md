# FRONTEND_GUIDELINES.md — 前端开发视角

## 分析快照

- 分支：main
- HEAD：a9cc17fd80648bfee0d0b677fa9ea91421f329fc
- 工作区状态：clean
- 子模块状态：无
- 分析时间：2026-07-18
- 分析范围：`web/`（Dashboard）、`ui-tui/`（TUI）、`apps/desktop/`（Electron）、`apps/bootstrap-installer/`（Tauri）、`apps/shared/`、`website/`（Docusaurus）、根 `package.json`、`eslint.config.shared.mjs`、`.prettierrc`、`tests-js/`
- 未覆盖范围：各前端组件逐个实现；样式系统 token 全量清单

## 证据分类

- Evidence：`package.json`、配置文件、入口文件、目录组织
- Inference：由重复代码模式推断的约定
- Unknown：仓库内无证据的规范（明确标注）

## 核心结论

[Evidence] Hermes 前端由**四个独立工程**组成，技术栈各异，共享极少：终端 UI（Ink/React）、Web Dashboard（React/Vite）、桌面壳（Electron）、安装器（Tauri）。共享 TS 库 `apps/shared` 仅被 web 与 desktop 消费。文档站为 Docusaurus。

[Evidence] 前端与 Python 后端的通信：HTTP/WS 到 FastAPI（dashboard/desktop）或 spawn Python sidecar（TUI）。前端**不直接**调用 agent，全部经后端。

---

## 1. 前端入口 / 页面 / 路由 / 布局

### 1.1 `web/`（Dashboard）
- 入口：`web/src/main.tsx` → `web/src/App.tsx`（react-router `Routes`/`NavLink` 外壳，页面懒加载）。
- 页面（`web/src/pages/`）：ConfigPage、DocsPage、EnvPage、FilesPage、SessionsPage、LogsPage、Chat、Git 等。
- 路由：react-router-dom 7（`App.tsx`）。
- 内嵌终端：xterm（经 `/api/pty` WS）。

### 1.2 `ui-tui/`（TUI）
- 入口：`ui-tui/src/entry.tsx`（shebang `node --max-old-space-size=8192`）。
- 主 hook：`ui-tui/src/app/useMainApp.ts`；app 壳 `ui-tui/src/app.tsx`。
- 渲染：**Ink 6**（终端 React 渲染器）+ 自维护 fork `@hermes/ink`（`ui-tui/packages/hermes-ink`）。
- 状态：**nanostores** + `@nanostores/react`。
- 打包：esbuild（`scripts/build.mjs`）；dev 用 `tsx --watch`。

### 1.3 `apps/desktop/`（Electron）
- 主进程：`apps/desktop/electron/main.ts`（`child_process.spawn`）。
- 后端拉起：`apps/desktop/electron/backend-command.ts:18-21` `hermes serve --host 127.0.0.1 --port 0`，legacy 回退 `dashboard --no-open`（`:30`）；Windows 树杀（`backend-child.ts`）。
- 渲染层：Vite/React，复用 dashboard。
- 打包：electron-builder（`dist:mac/win/linux`）。

### 1.4 `apps/bootstrap-installer/`（Tauri）
- 框架：Tauri 2 + Vite/React；Rust 壳 `src-tauri/`。
- 路由：`src/routes/`（welcome/progress/success/failure）。
- 职责：驱动 `install.ps1`/`setup-hermes.sh` 的原生安装器 UI。

### 1.5 `website/`
- Docusaurus 3.9（`docusaurus.config.ts`，`url: https://hermes-agent.nousresearch.com`，`baseUrl: /docs/`，i18n en + zh-Hans，theme-mermaid，search-local）。

---

## 2. 组件层级 / 目录组织 / 状态管理 / 数据获取

[Inference] 目录约定（由 `web/`、`ui-tui/` 推断）：
- `src/pages/`（页面）、`src/components/`（组件）、`src/lib/`（工具）、`src/routes/`（路由组件）。
- 状态：web 用组件局部 state + `@hermes/shared` 类型；TUI 用 nanostores。
- 数据获取：web 经 fetch/WS 到 FastAPI（`/api/*`、`/api/pty`、`/api/ws`）；TUI 经 `gatewayClient.ts` 与 Python sidecar。

[Evidence] API/IPC 封装：
- web：内联 fetch 到同源 FastAPI（dashboard 由 `hermes_cli/web_server.py` 同源提供）。
- TUI：`ui-tui/src/gatewayClient.ts`——`spawn(python, ['-m','tui_gateway.entry'])`（`:356`）+ 可选 sidecar WS 镜像（`connectSidecarMirror:268`）。slash 命令经 `setupHandoff.ts` 转交 `hermes …`。

---

## 3. 样式系统 / Design Token / 主题 / 响应式 / 可访问性 / 国际化 / 图标

| 项 | 状态 | Evidence |
| --- | --- | --- |
| 样式系统 | web 用 **Tailwind 4**；UI 库 `@nous-research/ui` | `web/package.json` |
| Design Token | **未发现统一的 Design Token 定义文件**（仓库内无 tokens.{js,ts,json} 之类） | [Evidence] 全仓搜索无命中 |
| 主题 | TUI 自维护 Ink fork 暗示终端配色控制；web 主题未见集中定义 | `ui-tui/packages/hermes-ink` |
| 响应式 | web 用 Tailwind 断点（推断） | `web/` Tailwind |
| 移动端适配 | **未发现移动端 Web 适配**（无移动专用入口/路由）；产品移动场景经 IM 网关而非移动 Web | [Evidence] 无移动端工程 |
| 可访问性 | 未见集中 a11y 规范/测试配置 | [Unknown] |
| 国际化 | Python 侧 `locales/*.yaml`（16 语言，仅 CLI 审批/网关 slash 静态文案）；website Docusaurus i18n en+zh-Hans；**前端 UI 文案未见统一 i18n 框架** | `locales/`、`website/i18n/` |
| 图标/静态资源 | `assets/`、各工程 `public/` | `apps/bootstrap-installer/public/` |

> [Evidence] 当前仓库未发现统一的 Design Token 定义。
> [Recommendation]（建议，非当前事实）：为 `@nous-research/ui` 引入集中 Design Token（颜色/间距/字号），统一 web 与 desktop 渲染。

---

## 4. 表单 / 校验 / 错误处理 / 加载状态 / 空状态

[Inference] 由 dashboard 各页面模式推断：
- 表单：受控组件 + 本地校验；后端校验在 FastAPI（`web_server.py` 各路由）。
- 错误处理：HTTP 错误状态展示；后端返回结构化错误。
- 加载/空状态：未见统一抽象（[Unknown]），多为页面内联。

---

## 5. 前端测试 / 构建 / 桌面/移动/浏览器差异 / 安全边界

- **测试**：
  - `ui-tui/`：vitest（`vitest.config.ts`），`ui-tui/src/__tests__/`（platform、bundleNoAsyncEsmDeadlock）。
  - `tests-js/`：3 个 `.test.ts`（assistant-ui-tap-compat、desktop-mac-entitlements、package-json-lazy-deps）。
  - CI：`js-tests.yml` per-workspace `npm run check`（typecheck+test）+ `npm run fix`。
- **构建**：web `vite build` → `hermes_cli/web_dist/`；TUI `node scripts/build.mjs` → `hermes_cli/tui_dist/`；desktop electron-builder；installer Tauri。
- **平台差异**：desktop 三平台（mac/win/linux）构建；Windows 有 entitlements/树杀特殊处理（`tests-js/desktop-mac-entitlements`、`apps/desktop/electron/backend-child.ts`）。
- **安全边界**：dashboard 本地鉴权（cookie/token，`hermes_cli/dashboard_auth/`）；CORS/security headers 中间件（`web_server.py`）；desktop 仅监听 `127.0.0.1`。

---

## 6. 代码组织约定（仓库内明确存在 vs 推断 vs 不存在）

| 约定 | 状态 | Evidence |
| --- | --- | --- |
| npm workspaces 聚合 | 明确存在 | 根 `package.json` |
| 共享 ESLint flat config | 明确存在 | `eslint.config.shared.mjs` |
| Prettier：no semi、single quote、120 cols | 明确存在 | `.prettierrc` |
| TS `check` = typecheck+test、`fix` = eslint --fix + prettier | 明确存在 | 各 workspace `package.json` scripts |
| 统一 Design Token | **不存在** | 全仓搜索无 |
| 统一 i18n 框架（前端） | **不存在** | 仅 locales + docusaurus i18n |
| 统一组件库跨 TUI/Web 共享 | **不存在**（TUI 用 Ink，Web 用 `@nous-research/ui`，互不共享） | 各 package.json |

---

## 已确认事实

- 四个独立前端工程 + 文档站，技术栈各异。
- 前端全部经后端（FastAPI 或 Python sidecar）与 agent 交互。
- 共享 ESLint/Prettier/workspace 约定。

## 合理推断

- 组件目录约定（pages/components/lib/routes）为团队共识。

## Unknown 与待验证事项

- Design Token、a11y、统一 i18n、加载/空状态抽象——仓库内无证据。

## 批判性评估

- 前端碎片化：Ink(TUI)/React-Web/Electron/Tauri 四套 UI 技术栈，组件/样式不可复用。
- 无统一 Design Token 与 i18n，多端一致性靠人工。

## 建设性改善建议

- [Recommendation] 为 `@nous-research/ui` 引入 Design Token 并在 web/desktop 强制消费。优先级：低；难度：中。
- [Recommendation] 引入前端 i18n 框架并与 Python `locales/` 对齐 key。优先级：低；难度：中。
- [Recommendation] 为 TUI 与 Web 抽象共享的 slash 命令/会话协议层（已有 `@hermes/shared`，可扩展）。优先级：低；难度：中。

## 主要证据索引

- `web/package.json`、`web/src/main.tsx`、`web/src/App.tsx`、`web/src/pages/`
- `ui-tui/package.json`、`ui-tui/src/entry.tsx`、`ui-tui/src/app.tsx`、`ui-tui/src/gatewayClient.ts:268,356`、`ui-tui/packages/hermes-ink`
- `apps/desktop/electron/{main,backend-command:18-21,backend-child}.ts`、`apps/desktop/package.json`
- `apps/bootstrap-installer/src-tauri/`、`apps/bootstrap-installer/src/routes/`
- `website/docusaurus.config.ts`、`website/i18n/`
- `eslint.config.shared.mjs`、`.prettierrc`、根 `package.json`、`tests-js/`
- `locales/`

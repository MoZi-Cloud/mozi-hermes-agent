# TECH_STACK.md — 技术栈与依赖

## 分析快照

- 分支：main
- HEAD：a9cc17fd80648bfee0d0b677fa9ea91421f329fc
- 工作区状态：clean（无未提交修改；分析内容等同 HEAD 纯净状态）
- 子模块状态：无子模块（不存在 .gitmodules）
- 分析时间：2026-07-18
- 分析范围：`pyproject.toml`、`uv.lock`、根级 `package.json` 与各前端 `package.json`、`Dockerfile`、`nix/`、源码 import 关系
- 未覆盖范围：`uv.lock` / `package-lock.json` 的逐条传递依赖；运行时尚未触发的可选 extra

## 证据分类

- Evidence：依赖声明 + 源码 import/初始化位置
- Inference：依赖的实际运行时角色（由调用路径推导）
- Unknown：某些 extra 是否在生产中被实际启用（取决于用户配置）

## 核心结论

[Evidence] Hermes Agent 是一个混合语言 monorepo：**Python（运行时核心）+ TypeScript/JavaScript（TUI、Dashboard、Desktop、Installer、文档站）+ Rust（Tauri 安装器原生壳）**。Python 包名为 `hermes-agent`，版本 `0.18.2`（`pyproject.toml:9-10`）。

[Evidence] 三个 Python 控制台入口（`pyproject.toml:307-310`）：
- `hermes` → `hermes_cli.main:main`（用户 CLI）
- `hermes-agent` → `run_agent:main`（一次性/datagen 运行）
- `hermes-acp` → `acp_adapter.entry:main`（ACP/编辑器集成）

---

## 1. 编程语言与用途

| 语言 | 用途 | Evidence |
| --- | --- | --- |
| Python ≥3.11,<3.14 | Agent 运行时、CLI、Gateway、Cron、MCP server/client、ACP server、工具实现、测试 | `pyproject.toml:24` (`requires-python`)；根级大型 `.py` 模块 |
| TypeScript / JavaScript | TUI（Ink）、Dashboard（React）、Desktop（Electron）、Installer（Tauri 渲染层）、文档站（Docusaurus）、共享库 | `ui-tui/package.json`、`web/package.json`、`apps/desktop/package.json`、`apps/bootstrap-installer/package.json`、`website/package.json` |
| Rust | Tauri 安装器原生壳（`src-tauri/`） | `apps/bootstrap-installer/src-tauri/Cargo.toml` |
| Shell | 安装脚本、Docker entrypoint、s6-overlay 服务定义 | `setup-hermes.sh`、`scripts/install.sh`、`docker/*.sh` |
| Nix | 可复现打包/devShell | `flake.nix`、`nix/*.nix` |
| Dockerfile | 容器镜像 | `Dockerfile`、`docker-compose.yml` |

---

## 2. Python 核心运行时依赖（关键项）

依赖采用**精确 pin（==X.Y.Z）**策略，注释明确说明这是为防范供应链攻击（`pyproject.toml:20-35`，引用 2026-05-12 mistralai 2.4.6 蠕虫事件）。

| 依赖 | 实际用途 | 初始化/调用位置 | 关键路径 | Evidence |
| --- | --- | --- | --- | --- |
| `openai==2.24.0` | 默认 LLM 客户端（OpenAI 兼容 chat completions） | `agent/chat_completion_helpers.py:317` `client.chat.completions.create` | 是（默认 transport） | `pyproject.toml:36` |
| `httpx[socks]==0.28.1` | HTTP 客户端（被 openai SDK 使用） | SDK 内部 | 是 | `pyproject.toml:42` |
| `pydantic==2.13.4` | 数据校验/序列化 | 多处模型定义 | 是 | `pyproject.toml:55`（注释说明 pydantic-core segfault） |
| `fastapi>=0.104,<1` + `uvicorn[standard]` + `python-multipart` | Dashboard 控制面 HTTP 服务 | `hermes_cli/web_server.py:264` `app = FastAPI(...)` | 是（dashboard/serve） | `pyproject.toml:93-99` |
| `prompt_toolkit==3.0.52` | 交互式 CLI 输入 | `cli.py`（`HermesCLI`） | 是（CLI 模式） | `pyproject.toml:58` |
| `rich==14.3.3` | 终端渲染 | `cli.py`、gateway | 是 | `pyproject.toml:41` |
| `fire==0.7.1` | （声明存在；用于一次性 CLI 解析） | `run_agent.py:main` 参数风格 | 否（仅 datagen） | `pyproject.toml:40` |
| `croniter==6.0.0` | cron 表达式解析 | `cron/scheduler.py` | 是（gateway 每 60s tick） | `pyproject.toml:61` |
| `jinja2==3.1.6` | 模板渲染（system prompt、skill 模板） | `agent/prompt_builder.py`、skill 渲染 | 是 | `pyproject.toml:53` |
| `ruamel.yaml` / `pyyaml` | config.yaml 读写 | `hermes_cli/config.py` | 是 | `pyproject.toml:51-52` |
| `tenacity==9.1.4` | 重试装饰器 | 多处（API 调用、写库） | 是 | `pyproject.toml:43` |
| `websockets==15.0.1` | 浏览器 CDP 监管 + dashboard WS | `tools/browser_*`、`web_server.py` `/api/pty` | 是 | `pyproject.toml:118` |
| `psutil==7.2.2` | 跨平台进程/PID 管理 | gateway、cron、`hermes_cli/psutil_android.py` | 是 | `pyproject.toml:113` |
| `Pillow==12.2.0` | 视觉工具图片缩放恢复 | 视觉嵌入路径 | 是（视觉工具） | `pyproject.toml:139` |
| `PyJWT[crypto]==2.13.0` + `cryptography==46.0.7` | Skills Hub GitHub App JWT 鉴权 / WeCom 加解密 | skills hub、wecom | 条件 | `pyproject.toml:89,97` |
| `Markdown==3.10.2` | Markdown→HTML（消息投递） | gateway 投递、`tools/send_message_tool.py` | 是（投递渲染） | `pyproject.toml:83` |
| `python-dotenv` | `.env` 加载 | `hermes_cli/env_loader.py` | 是 | `pyproject.toml:39` |
| `ptyprocess` / `pywinpty` | PTY（非 Windows / Windows） | `hermes_cli/pty_*.py`、tui_gateway | 是（终端工具） | `pyproject.toml:101-103` |

### 2.1 可选 extra（按需 lazy-install 或 extra-install）

[Inference] extra 分两类：(a) 进程内 provider 后端（`anthropic`、`exa`、`firecrawl`、`fal`、`edge-tts`、`mistral`、`bedrock`、`vertex`、`azure-identity` 等）；(b) 平台/通道后端（`messaging`、`slack`、`matrix`、`teams`、`dingtalk`、`feishu`、`wecom`、`sms`、`homeassistant`、`google` 等）。

[Evidence] 重型依赖（如 `mautrix[encryption]`、`mistralai`、`supermemory`、`mem0ai`、`honcho-ai`）通过 `tools/lazy_deps.py` 在首次使用时安装，**不进入默认安装**（`pyproject.toml` 注释多次引用 `tools/lazy_deps.py`）。

[Unknown] 生产环境实际启用了哪些 extra——取决于用户 `config.yaml`，仓库静态扫描无法确认。

### 2.2 消息平台 SDK（messaging/slack/matrix/teams 等）

| 依赖 | 用途 | Evidence |
| --- | --- | --- |
| `python-telegram-bot[webhooks]==22.6` | Telegram | `pyproject.toml` messaging extra |
| `discord.py[voice]==2.7.1` | Discord | 同上 |
| `slack-bolt==1.29.0` / `slack-sdk==3.43.0` | Slack（Socket Mode） | 同上；`plugins/platforms/slack/adapter.py:23` |
| `mautrix[encryption]==0.21.0` | Matrix（端到端加密） | matrix extra |
| `microsoft-teams-apps==2.0.13.4` | MS Teams | teams extra |
| `dingtalk-stream` / `lark-oapi` | 钉钉 / 飞书 | dingtalk/feishu extra |
| `aiohttp==3.14.1` | 异步 HTTP（被多平台使用，CVE 修复 pin） | 多 extra 重复 pin |

---

## 3. 前端 / 桌面 / 文档技术栈

[Inference] 仓库根 `package.json` 使用 **npm workspaces**，聚合 `apps/*`、`ui-tui`、`ui-tui/packages/*`、`web`、`tests-js`。

| 子工程 | 框架 | 入口 | Evidence |
| --- | --- | --- | --- |
| `ui-tui/`（hermes-tui） | **React 19 + Ink 6**（终端 React 渲染）+ nanostores + 自维护 Ink fork `@hermes/ink`；esbuild 打包 | `ui-tui/src/entry.tsx` | `ui-tui/package.json` |
| `web/` | **React 19 + Vite 8 + react-router 7 + Tailwind 4**；UI 库 `@nous-research/ui`；xterm 终端；共享库 `@hermes/shared` | `web/src/main.tsx` → `App.tsx` | `web/package.json` |
| `apps/desktop/` | **Electron + Vite/React** 渲染层；electron-builder 打包 | `apps/desktop/electron/main.ts` | `apps/desktop/package.json` |
| `apps/bootstrap-installer/` | **Tauri 2 + Vite/React**；Rust 原生壳 | `src-tauri/` | `apps/bootstrap-installer/src-tauri/Cargo.toml` |
| `apps/shared/` | 共享 TS 库（被 web、desktop 消费） | — | `apps/shared/package.json` |
| `website/` | **Docusaurus 3.9**（theme-mermaid、search-local） | `website/docusaurus.config.ts` | `website/package.json` |

[Evidence] 前端构建产物被打包进 Python wheel：`hermes_cli/web_dist/**` 与 `hermes_cli/tui_dist/**`（`pyproject.toml:339` `[tool.setuptools.package-data]`）。Dashboard 由 `hermes_cli/web_server.py` 以 `StaticFiles` 挂载（`web_server.py:101,126,2016`）。

---

## 4. 构建工具 / 包管理器 / workspace

| 工具 | 用途 | Evidence |
| --- | --- | --- |
| **uv**（Astral） | Python 包管理与锁文件；CI 用 `uv sync --locked` | `uv.lock`；`pyproject.toml:20`；`.github/workflows/tests.yml` |
| setuptools ≥77 | 构建后端（`build-system`） | `pyproject.toml:6-8`；`setup.py` 定制 `build`/`egg_info`（只读源码树保护） |
| npm（Node 22） | JS workspace 管理 | 根 `package.json`；CI `js-tests.yml` |
| Vite / esbuild | 前端/TUI 打包 | 各 `package.json` 的 `build` 脚本 |
| electron-builder | Desktop 打包 | `apps/desktop/package.json` |
| Tauri CLI | Installer 打包 | `apps/bootstrap-installer/src-tauri/tauri.conf.json` |

---

## 5. 数据存储 / 数据访问

[Evidence] 无传统 ORM。持久化以 **SQLite + FTS5** 为主，辅以 JSON 文件：
- 会话/消息/搜索：`hermes_state.py` `SessionDB`（`hermes_state.py:971`；FTS5 虚拟表，`hermes_state.py:5,11`）。
- Cron 作业：`~/.hermes/cron/jobs.json`（`cron/jobs.py:4,71`，跨进程文件锁）。
- 异步委派：SQLite `state.db` 中的 `async_delegations` 表（`tools/async_delegation.py:93`）。
- Kanban、dashboard 响应存储等另有 SQLite（`hermes_cli/kanban_db.py`、`gateway/platforms/api_server.py:ResponseStore:404`）。

---

## 6. API 与通信协议

- **OpenAI 兼容 chat completions**（默认 transport，`agent/transports/chat_completions.py`）。
- **Anthropic Messages API**（原生 transport，`agent/transports/anthropic.py`）。
- **Bedrock / Vertex / Gemini 原生 / Codex（含 codex_app_server 子进程 JSON-RPC over stdio）**（`agent/transports/`）。
- **MCP**（Model Context Protocol）：Hermes 既作客户端（`tools/mcp_tool.py`），也作服务端（`mcp_serve.py`，stdio）。
- **ACP**（Agent Client Protocol，JSON-RPC）：`acp_adapter/server.py`，供 Zed 等编辑器驱动。
- **WebSocket**：dashboard `/api/pty`、`/api/ws`、`/api/console`（`hermes_cli/web_server.py:16186,16363,15836`）；tui_gateway `/api/ws`（`tui_gateway/ws.py:19`）。
- **aiohttp**：gateway 内置 OpenAI 兼容 HTTP API（`gateway/platforms/api_server.py:917`，非 FastAPI）。

---

## 7. CI/CD / 容器 / 部署

详见 `测试与CI.md`。要点：

- CI 编排：`.github/workflows/ci.yml`（detect → 扇出子 workflow → `all-checks-pass` 聚合门）。
- 发布：CalVer tag `v20*` 触发 `upload_to_pypi.yml`（OIDC trusted publishing + Sigstore 签名）。
- 容器：`Dockerfile` 多阶段（uv + node + debian:13.4 + **s6-overlay v3** 监管，`/init` 为 PID 1）；`docker-compose.yml` 两服务（gateway + dashboard）。
- 分发：PyPI、Homebrew tap（`packaging/homebrew/`）、Nix flake、安装脚本（`setup-hermes.sh`、`scripts/install.{sh,ps1,cmd}`）。

---

## 8. 日志 / 监控 / 代码生成

| 项 | 状态 | Evidence |
| --- | --- | --- |
| 日志 | `hermes_logging.py:setup_logging`（259），队列 handler、按会话上下文过滤、跨进程文件锁；Windows 用 `concurrent-log-handler` 轮转 | `hermes_logging.py`；`pyproject.toml` Windows marker |
| 监控/可观测 | `plugins/observability/` 插件；`agent/stream_diag.py`、`agent/aux_accounting.py`、用量追踪 | 插件目录 |
| Trajectory 生成/压缩 | `batch_runner.py`、`trajectory_compressor.py`（训练数据生成） | 根级模块 |
| 代码生成 | LSP 集成（`agent/lsp/`）；无大规模手写→生成代码 | `agent/lsp/` |

---

## 9. Feature flag / 平台标记

[Evidence] 大量 `sys_platform` marker（Windows 专属 `concurrent-log-handler`、`pywinpty`、`tzdata`；非 Windows `ptyprocess`）。运行时 feature 由 `config.yaml` 控制（provider、toolsets、plugins.enabled、memory.provider、cron.provider 等），见 `hermes_cli/config.py`。环境变量 `HERMES_*` 大量存在（`HERMES_SAFE_MODE`、`HERMES_PLUGINS_DEBUG`、`HERMES_ENABLE_PROJECT_PLUGINS`、`HERMES_TUI_GATEWAY_URL` 等）。

---

## 10. Git 子模块 / vendored source

[Evidence] 无 Git 子模块（无 `.gitmodules`）。无显式 `vendor/` 目录。`agent/lsp/`、`agent/pet/` 为项目自有子模块。第三方源码以 PyPI/npm 依赖形式引入，未 vendored 进仓库。

---

## 已确认事实

- Python + TS + Rust + Nix + Docker 多语言 monorepo；npm workspaces 聚合前端。
- 依赖精确 pin，按 extra/lazy 分层安装，供应链加固（Sigstore、OSV、supply-chain-audit workflow）。
- 三个 Python 入口、两个主要 HTTP 服务（FastAPI dashboard + aiohttp gateway API）。

## 合理推断

- `tools/lazy_deps.py` + extra 机制构成"按需安装"体系，使默认安装体积可控。
- 前端构建产物随 Python wheel 分发，确保 `hermes dashboard/serve` 开箱即用。

## Unknown 与待验证事项

- 各 extra 在真实部署中的启用比例（依赖用户配置）。
- `fire` 是否仍在主路径使用（仅见于 `run_agent.py:main` 风格，可能为遗留）。

## 批判性评估

- 顶层存在大量"巨型文件"（`cli.py` 16465 行、`run_agent.py` 6372 行、`hermes_state.py` 7533 行、`agent/auxiliary_client.py` 7881 行）。`run_agent.py` 的 `AIAgent` 已大量改为转发到 `agent/` 子模块——处于"重构进行中"状态。
- 同时存在 FastAPI（dashboard）与 aiohttp（gateway API）两套 HTTP 栈，且 Teams 适配器内部还桥接了一个 FastAPI 实例到 aiohttp（`plugins/platforms/teams/adapter.py:357`）——技术栈不统一。

## 建设性改善建议

- [Recommendation] 继续推进 `run_agent.py` → `agent/` 的拆分，直至 `AIAgent` 成为纯状态持有者，消除"转发方法"双重维护面。优先级：中；难度：高；依据：`run_agent.py` 已有大量 forwarder（4685/4981/5204/5545/5915/6079）。
- [Recommendation] 统一 HTTP 服务层（dashboard 与 gateway API 共用同一 ASGI 框架），消除 Teams 适配器中 aiohttp↔FastAPI 桥接。优先级：低；难度：高。

## 主要证据索引

- `pyproject.toml:8-10,20-35,143-306,307-310,339,359`
- `setup.py:78-87`
- `hermes`（launcher）
- `hermes_cli/web_server.py:264`
- `agent/chat_completion_helpers.py:317`
- `agent/transports/`（各 transport）
- `ui-tui/package.json`、`web/package.json`、`apps/desktop/package.json`、`website/docusaurus.config.ts`
- `Dockerfile`、`flake.nix`

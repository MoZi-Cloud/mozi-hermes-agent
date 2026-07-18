# APP_FLOW.md — 用户视角应用流程

## 分析快照

- 分支：main
- HEAD：a9cc17fd80648bfee0d0b677fa9ea91421f329fc
- 工作区状态：clean
- 子模块状态：无
- 分析时间：2026-07-18
- 分析范围：`README.md`、`hermes_cli/main.py`、`hermes_cli/setup.py`、`cli.py`、`gateway/run.py`、`tui_gateway/`、`apps/desktop/electron/`、`hermes_cli/subcommands/`
- 未覆盖范围：真实安装器交互（`setup-hermes.sh` 未逐行执行）

## 证据分类

- Evidence：命令/子命令注册与处理函数
- Inference：由命令链推导的用户可感知流程
- Unknown：安装器在不同 OS 的实际副作用（未动态执行）

## 核心结论

[Evidence] Hermes 提供**两条主入口**：(1) `hermes`（交互式 CLI/TUI 对话）；(2) `hermes gateway`（消息网关，从 IM 平台对话）。两者共享同一 agent 运行时与多数 slash 命令（`README.md:145-159`）。

[Evidence] 安装后无强制登录。**当前源码未发现用户账户/登录认证流程**作为 agent 运行的前置条件——认证仅针对**外部 LLM provider 的 API key / OAuth**（`hermes_cli/auth*.py`、`hermes_cli/portal_cli.py`、`hermes_cli/copilot_auth.py`、`hermes_cli/memory_oauth.py`）。Gateway 有**设备配对（DM pairing）**机制（`gateway/pairing.py`、`hermes_cli/pairing.py`）用于授权谁可通过 IM 与 bot 对话。

---

## 1. 安装 / 首次运行 / 初始化配置

[Evidence] 安装路径：
- Linux/macOS/WSL/Termux：`curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash`（`README.md:40`）
- Windows PowerShell：`iex (irm .../install.ps1)`（`README.md:50`）
- 安装器负责：uv、Python 3.11、Node.js、ripgrep、ffmpeg、（Windows）便携 Git Bash（`README.md:53-55`）。

[Evidence] 安装后执行：
```
source ~/.bashrc
hermes              # start chatting!
```
首次 `hermes` 触发自愈/初始化逻辑（`hermes_cli/main.py:13166 main()` 中"self-heal interrupted installs"）。

[Evidence] 配置向导：`hermes setup`（全量）或 `hermes setup --portal`（Nous Portal OAuth 一键，`README.md:134-137`；实现 `hermes_cli/setup.py`、`hermes_cli/portal_cli.py`）。子命令注册见 `hermes_cli/subcommands/`。

[Evidence] 配置存储：`~/.hermes/config.yaml`（`hermes_cli/config.py` `load_config`/`get_config_path`），含 provider/model/toolsets/platforms/cron/memory 等。

---

## 2. 用户登录 / 无登录模式 / 权限

[Evidence] **当前源码未发现"用户登录或账户认证"流程**作为产品前置。所谓"认证"全部是：
- 为 LLM provider 配置 API key 或完成 OAuth（`hermes_cli/auth_commands.py`、`hermes_cli/copilot_auth.py`、`hermes_cli/nous_auth_keepalive.py`、`hermes_cli/memory_oauth.py`）。
- Nous Portal 订阅登录（`hermes_cli/portal_cli.py`）——可选。
- Dashboard 本地鉴权（`hermes_cli/dashboard_auth/`、`hermes_cli/dashboard_register.py`）——本地 Web 令牌。
- Gateway **DM pairing**：决定哪些 IM 用户可与 bot 对话（`gateway/pairing.py`、`hermes_cli/pairing.py`、`gateway/authz_mixin.py`）。

[Inference] 产品定位为单租户自托管，因此无多租户账户体系。

---

## 3. 启动对话（CLI / TUI）

[Evidence] `hermes` → `hermes_cli.main:main`（`main.py:13166`）→ 解析子命令 → 默认 `cmd_chat`（`main.py:2255`）。
- `_resolve_use_tui`（`main.py:2212`）决定 TUI vs 经典 CLI。
- TUI：`_launch_tui`（`main.py:2006`）拉起 Node TUI（`ui-tui/`）；TUI 通过 `gatewayClient.ts:356` `spawn(python, ['-m','tui_gateway.entry'])` 启动 Python sidecar（`tui_gateway/`），sidecar 在进程内驱动 agent。
- 经典 CLI：`from cli import main as cli_main; cli_main(...)`（`main.py:2436/2461`）→ `HermesCLI`（`cli.py:3657`）。

[Evidence] 共享 slash 命令（`README.md:148-158`）：`/new`、`/reset`、`/model`、`/personality`、`/retry`、`/undo`、`/compress`、`/usage`、`/insights`、`/skills`、`/stop`、`/<skill-name>` 等。实现：`hermes_cli/cli_commands_mixin.py`、`hermes_cli/commands.py`（补全/建议）。

### 3.1 一次对话的源码对应

```
用户输入 → cli.py / tui_gateway 捕获
→ AIAgent.run_conversation (run_agent.py:6079 → agent/conversation_loop.py:537)
→ build_turn_context (agent/turn_context.py)
→ while 循环 (conversation_loop.py:661)：LLM 调用 ↔ 工具执行
→ 流式回调渲染到终端 (stream_callback / step_callback)
→ 最终回复 + 写入 SessionDB (hermes_state.py)
```

---

## 4. 启动消息网关（IM 对话）

[Evidence] 流程（`README.md:148-149`）：
```
hermes gateway setup    # 配置平台 token、允许的用户、工作目录
hermes gateway start    # 启动网关进程
# 然后在 IM 平台给 bot 发消息
```
实现：`hermes_cli/gateway.py` → `gateway/run.py:start_gateway`（`run.py:21795 asyncio.run`）。

[Evidence] 网关内 inbound 管道（`gateway/run.py`）：
1. 平台适配器（`BasePlatformAdapter.handle_message`，`base.py:4676`）触发 `MessageEvent`。
2. `GatewayRunner._handle_message`（`run.py:9235`）七步管道：授权（authz）→ slash 命令 → 中断检查 → 会话定位 → 上下文 → 运行 agent → 返回。
3. `_handle_message_with_agent`（`run.py:11187`）→ `_run_agent`（`run.py:17586`）→ `_run_agent_inner`（`run.py:17736`）构造 `AIAgent` 并驱动。
4. 回复经 `adapter.send(chat_id, content, ...)`（`base.py:2954`）投递回 IM。

[Evidence] 平台特定状态命令：CLI 无；IM 有 `/status`、`/sethome`、`/platforms`（`README.md:157`）。

---

## 5. 核心任务流程

### 5.1 选模型 / 配工具 / 配技能
- `hermes model` / `/model`（`hermes_cli/model_switch.py`、`hermes_cli/models.py`）
- `hermes tools`（`hermes_cli/tools_config.py`、`hermes_cli/toolset_validation.py`）
- `/skills`、`/<skill-name>`（`tools/skills_tool.py`）

### 5.2 数据保存 / 持久化
[Evidence] 每轮对话自动写入 `SessionDB`（SQLite，`hermes_state.py:971`）；记忆由 `agent/memory_manager.py` + memory provider 持久化；技能写入 `~/.hermes/skills/`。

### 5.3 数据导入 / 导出 / 迁移
- OpenClaw 迁移：`hermes claw migrate [--dry-run] [--preset ...]`（`hermes_cli/claw.py`；`README.md:195-213`）。
- 会话导出：`hermes_cli/session_export*.py`（MD/HTML）。
- 备份：`hermes_cli/backup.py`。

### 5.4 定时任务（cron）
- 通过 `cronjob` 工具或 `hermes cron` 创建（`tools/cronjob_tools.py`、`hermes_cli/cron.py`）。
- gateway 每 60s 调 `cron.scheduler.tick`（`cron/scheduler.py:3801`）。
- 投递：`gateway/delivery.py DeliveryRouter.deliver`（`delivery.py:246`）路由到任意平台。

### 5.5 多工作区 / Profile 切换
[Evidence] Profile 系统：`hermes_cli/profiles.py`、`hermes_cli/projects_cmd.py`、`gateway/profile_routing.py`。每个 profile 独立 config / cron / skills 目录（`~/.hermes/profiles/<p>/`）。

---

## 6. 错误提示 / 异常恢复

[Evidence]
- 工具错误不抛给模型，转为 `{"error": ...}` JSON（`tools/registry.py:614 dispatch`）。
- LLM 调用有 fallback 链（`agent/chat_completion_helpers.py:1407 try_activate_fallback`）与重试（`tenacity`）。
- gateway 异常处理器（`gateway/run.py:276 _gateway_loop_exception_handler`）防止单条消息崩溃整个网关。
- 崩溃恢复：会话状态持久化 + `recover_abandoned_delegations`（`tools/async_delegation.py:219`）。
- 诊断：`hermes doctor`（`hermes_cli/doctor.py`）。

---

## 7. 退出 / 再次启动 / 升级 / 数据恢复

- 退出：CLI `Ctrl+C` 或 `/stop`；gateway 优雅关闭（`gateway/run.py` 生命周期 + `gateway/shutdown_forensics.py`）。
- 升级：`hermes update`（`hermes_cli/main.py` 子命令，更新 managed install）。
- 再次启动：从 `SessionDB` 恢复历史（`--continue`/`--resume`，`cmd_chat` 解析 session id）。
- 数据恢复：备份/导出文件 + SQLite WAL。

---

## 已确认事实

- 两入口（CLI/TUI、gateway），共享 agent 运行时与 slash 命令。
- 无"用户登录"前置；认证针对外部 provider API key/OAuth 与 gateway DM pairing。
- 安装器、setup 向导、profile、cron、迁移、导出均有实现。

## 合理推断

- 用户感知的"登录"实际是 provider OAuth/Portal 订阅，而非产品账户。

## Unknown 与待验证事项

- 安装器在不同 OS 的实际网络下载与权限申请细节（未动态运行）。
- Desktop 首次启动的引导流程（Electron 拉起 `hermes serve`）的 UI 细节未逐屏确认。

## 批判性评估

- README 将"安装即用"表述得很轻量，但实际依赖链（Node、Electron、可选 extra）较重。
- 无账户体系意味着安全边界完全依赖本地配置与 DM pairing，多用户场景需谨慎。

## 建设性改善建议

- [Recommendation] 在 `hermes setup` 流程中显式提示"无产品账户；认证仅用于 provider"，减少用户困惑。优先级：低；难度：低。

## 主要证据索引

- `README.md:40-66,107-118,145-159,195-213`
- `hermes_cli/main.py:13166(main),2255(cmd_chat),2006(_launch_tui),2212(_resolve_use_tui)`
- `gateway/run.py:9235(_handle_message),11187,17586,17736,21795`
- `gateway/platforms/base.py:4676(handle_message),2954(send)`
- `tui_gateway/`、`apps/desktop/electron/backend-command.ts:18-21`
- `hermes_cli/{setup,portal_cli,claw,pairing,profiles,doctor,backup,session_export*}.py`
- `gateway/{pairing,authz_mixin,delivery,profile_routing}.py`

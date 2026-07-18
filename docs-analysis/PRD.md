# PRD.md — 产品需求（基于实际实现）

## 分析快照

- 分支：main
- HEAD：a9cc17fd80648bfee0d0b677fa9ea91421f329fc
- 工作区状态：clean
- 子模块状态：无
- 分析时间：2026-07-18
- 分析范围：`README.md`（及多语言译本）、根级命令清单、`hermes_cli/subcommands/`、`gateway/`、`agent/`、`tools/`、`skills/`、`plugins/`、`cron/`、`acp_adapter/`
- 未覆盖范围：线上 SaaS（Nous Portal / Tool Gateway）后端——不在仓库内

## 证据分类

- Evidence：源码中可直接运行/调用的能力
- Inference：由代码结构合理推断的产品定位
- Unknown：仅文档/官网声称但仓库内无对应实现的能力

## 核心结论

[Evidence] 产品名 **Hermes Agent**，由 **Nous Research** 出品（`pyproject.toml:11`），定位为"自我改进的 AI agent"——核心卖点是内置学习闭环（从经验创建技能、使用中改进技能）、跨会话记忆、跨平台消息网关、可调度自动化、可在任意基础设施运行（`README.md:19-31`）。

[Inference] 该产品同时面向三类用户：(1) 终端高级用户/开发者（CLI/TUI）；(2) 希望从即时通讯软件对话的用户（Telegram/Discord/Slack 等 gateway）；(3) 研究/训练数据生成场景（batch_runner、trajectory_compressor）。

---

## 1. 项目定位 / 目标用户 / 解决的问题

| 维度 | 内容 | Evidence |
| --- | --- | --- |
| 定位 | 自托管、模型无关、自带学习闭环的 AI agent | `README.md:19` |
| 目标用户 | 开发者、研究者、希望"随时随地"通过 IM 与 agent 对话的用户 | `README.md:24-31` |
| 解决的问题 | 模型锁定、会话断层、技能不可累积、只能在本地运行 | 同上 |
| 适用场景 | 编码/文件操作、定时任务、消息机器人、训练数据生成、桌面控制 | 工具集与平台适配器清单 |
| 产品边界 | **不是**：模型本身、云托管平台（Nous Portal 是独立 SaaS）、移动原生 App | 仓库内无移动 App、无自研模型 |

---

## 2. 核心功能实现状态矩阵

| 功能 | 文档声明 | 源码状态 | 运行时入口 | 测试证据 | 最终判断 |
| --- | --- | --- | --- | --- | --- |
| 交互式 CLI（slash 命令、流式、历史） | ✓ | `cli.py` `HermesCLI`（3657）、`main`（15950）、`run`（13322） | `hermes`（无 TUI 时） | `tests/cli/` 90 文件 | 已完整实现 |
| 终端 UI（TUI） | ✓ "real terminal interface" | `ui-tui/`（Ink/React），`tui_gateway/` sidecar | `hermes`（`_launch_tui`，`main.py:2006`） | `tests/tui_gateway/` 33 文件 | 已完整实现 |
| 模型无关 / 切换 | ✓ "Use any model" | `providers/` + `plugins/model-providers/` 32 个；`agent/transports/` | `hermes model` / `/model` | `tests/providers/` | 已完整实现 |
| 工具系统（40+ 工具） | ✓ | `tools/*.py` 自注册到 `registry`；`toolsets.py` | agent loop `tool_executor` | `tests/tools/` 323 文件 | 已完整实现 |
| 工具集（toolsets） | ✓ | `toolsets.py:96 TOOLSETS`；`toolset_distributions.py` | `hermes tools` | `tests/` | 已完整实现 |
| 技能系统（procedural memory） | ✓ agentskills.io | `tools/skills_tool.py`、`tools/skill_manager_tool.py`、`skills/` 453 文件、`optional-skills/` | `/skills`、`skill_view` | `tests/skills/` | 已完整实现 |
| 技能自动创建/自我改进 | ✓ "creates skills from experience" | `agent/curator.py`、`agent/background_review.py`、`skill_manage` 工具 | curator 定时 + 后台评审 | `tests/agent/` | 已实现，**默认关闭**（`agent/curator.py:74,205-211`） |
| 记忆 / 用户建模 | ✓ Honcho 等 | `plugins/memory/` 8 个 provider（honcho/mem0/supermemory/hindsight/...） | `memory.provider` config | `tests/honcho_plugin/` 等 | 已完整实现 |
| 跨会话搜索（FTS5） | ✓ | `hermes_state.py` `SessionDB` FTS5 虚拟表 | `/insights`、session 搜索 | `tests/hermes_state/` | 已完整实现 |
| 消息网关（多平台） | ✓ Telegram/Discord/Slack/WhatsApp/Signal/Email | `gateway/` + `plugins/platforms/` 20 适配器 | `hermes gateway` | `tests/gateway/` 492 文件 | 已完整实现 |
| 定时自动化（cron） | ✓ | `cron/scheduler.py`、`cron/jobs.py`、`tools/cronjob_tools.py` | gateway 每 60s tick；`hermes cron` | `tests/cron/` 35 文件 | 已完整实现 |
| 子 agent / 委派 / 并行 | ✓ "Delegates and parallelizes" | `tools/delegate_tool.py`、`tools/async_delegation.py` | `delegate_task` 工具 | `tests/tools/` | 已完整实现 |
| RPC 脚本执行 | ✓ "call tools via RPC" | codex_app_server transport（JSON-RPC over stdio）；MCP server | `mcp_serve.py` | `tests/` | 已实现（通过 codex_app_server / MCP） |
| 终端后端（6 种） | ✓ local/Docker/SSH/Singularity/Modal/Daytona | `tools/environments/` 全部存在 + `managed_modal` | `execute_code` 工具 | `tests/tools/` | 已完整实现 |
| Serverless 持久化（Daytona/Modal） | ✓ "hibernates when idle" | `tools/environments/{daytona,managed_modal}.py`；`gateway/scale_to_zero.py` | config | `tests/` | 已实现（含 scale_to_zero 机制） |
| 训练数据生成 | ✓ "Batch trajectory generation" | `batch_runner.py`、`trajectory_compressor.py`、`toolset_distributions.py` | `hermes-agent` / batch | `tests/` | 已完整实现 |
| MCP 集成 | ✓ | `tools/mcp_tool.py`（客户端）、`mcp_serve.py`（服务端）、`optional-mcps/` 目录 | `hermes mcp` | `tests/` | 已完整实现 |
| 桌面应用 | ✓ "Hermes Desktop" | `apps/desktop/`（Electron） | `hermes serve`（被 Electron 拉起） | `tests-js/desktop-*` | 已完整实现 |
| 浏览器控制 / 计算机使用 | （文档较少） | `tools/browser_*`、`tools/computer_use/`、`plugins/browser` | 工具 | `tests/computer_use/` | 已实现 |
| OpenAI 兼容 API | （未在 README 主表强调） | `gateway/platforms/api_server.py` `/v1/chat/completions` 等 | gateway `api_server` platform | `tests/gateway/` | 已完整实现 |
| Nous Portal / Tool Gateway | ✓ OAuth 一键 | 客户端代码 `hermes_cli/portal_cli.py`、`hermes setup --portal` | `hermes setup --portal` | `tests/` | **客户端已实现；SaaS 后端不在仓库（Unknown）** |
| OpenClaw 迁移 | ✓ | `hermes_cli/claw.py`、`skills/openclaw-migration` | `hermes claw migrate` | `tests/` | 已完整实现 |

---

## 3. 用户角色 / 输入 / 输出 / 外部依赖

- **角色**：单用户为主（自托管），但支持 gateway 多聊天/多用户路由（`gateway/profile_routing.py`、`gateway/platform_registry.py`）。
- **输入**：自然语言文本（CLI/TUI/IM）、语音（STT，`plugins/memory`... 实为 `agent/transcription_registry.py`）、图像（视觉工具）、文件。
- **输出**：文本回复、工具执行结果、文件改动、定时报告投递到 IM 平台、语音（TTS）、图像/视频生成。
- **外部依赖**：LLM provider API（必须）、可选的搜索/图像/TTS/浏览器后端、可选的 IM 平台 token、可选的云沙箱（Modal/Daytona）。

---

## 4. 当前实现产品 vs 文档愿景 vs 推断方向

### 当前已实现产品
一个模型无关、自托管、多入口（CLI/TUI/Dashboard/Desktop/IM/OpenAI-API/ACP/MCP）的 agent 运行时，带工具/工具集/技能/记忆/cron/委派/多沙箱/训练数据生成。

### README/docs 描述的愿景
- "self-improving"——自我改进闭环。[Evidence] 实现存在但 **默认关闭**（`agent/curator.py:74 is_enabled` 默认行为 + `:205-211` consolidate 默认关闭）。README 措辞暗示默认开启，与源码有出入。
- Nous Portal / Tool Gateway SaaS——[Unknown] 仓库内只有客户端，后端不在仓库。

### 源码可推断的发展方向
- 重型重构：`run_agent.py` 正被拆入 `agent/`（大量 forwarder 方法）。
- 供应链安全强化（精确 pin、Sigstore、OSV、supply-chain-audit）。
- 多平台扩张（20 个消息适配器、32 个模型 provider）。

### 改善建议（见各专项文档）

---

## 5. 非目标 / 当前限制 / 产品风险 / 数据与隐私边界

- **非目标**：不提供自有模型；不托管用户数据（自托管）；非移动原生 App。
- **限制**：默认安装不含重型依赖（需 extra/lazy）；TUI 需 Node；Desktop 需 Electron；ACP/codex 集成依赖外部二进制。
- **产品风险**：
  - agent 可执行任意 shell/文件操作（`tools/terminal_tool.py`、`tools/file_tools.py`）——虽有审批机制（`tools/approval.py`，173KB），但默认策略影响安全边界。
  - 多平台 token 与凭据集中存储于 `~/.hermes`。
- **数据与隐私边界**：会话/记忆存本地 SQLite/JSON；与 LLM provider 的通信是数据出站点；记忆 provider（mem0/supermemory/honcho）为可选云端。

---

## 已确认事实

- README 主要功能声明均有源码支撑（见矩阵"最终判断"列）。
- 三入口、多前端、多平台、多 provider、多沙箱均真实存在并被入口调用。

## 合理推断

- "self-improving"是核心叙事但实际默认关闭，属"能力具备但保守交付"。
- 产品同时服务终端用户与研究者两类人群。

## Unknown 与待验证事项

- Nous Portal / Tool Gateway 的服务端实现不在仓库。
- 各可选 provider/平台在真实部署中的启用比例。

## 批判性评估

- README 对"self-improving / autonomous skill creation"的表述偏强于源码默认行为（默认关闭）。
- "Runs anywhere"基本属实，但 TUI/Desktop/ACP 各有额外运行时依赖，非纯 Python 单二进制。

## 建设性改善建议

- [Recommendation] 在 README 显式标注"自主技能创建默认关闭，需显式启用 curator"，避免预期偏差。优先级：低；难度：低；依据：`agent/curator.py:74,205-211`。

## 主要证据索引

- `README.md:19-31,107-118,143-159`
- `pyproject.toml:11`
- `hermes_cli/main.py:2006(_launch_tui),2255(cmd_chat)`
- `tools/environments/`、`gateway/platforms/`、`plugins/platforms/`、`plugins/memory/`、`plugins/model-providers/`
- `agent/curator.py:74`、`agent/background_review.py`
- `gateway/platforms/api_server.py:917`
- `hermes_cli/portal_cli.py`、`hermes_cli/claw.py`

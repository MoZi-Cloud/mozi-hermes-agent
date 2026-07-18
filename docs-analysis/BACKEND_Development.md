# BACKEND_Development.md — 后端开发视角

## 分析快照

- 分支：main
- HEAD：a9cc17fd80648bfee0d0b677fa9ea91421f329fc
- 工作区状态：clean
- 子模块状态：无
- 分析时间：2026-07-18
- 分析范围：Python 后端全部（`agent/`、`run_agent.py`、`hermes_cli/`、`gateway/`、`tools/`、`cron/`、`acp_adapter/`、`hermes_state.py`、`hermes_logging.py`、`hermes_cli/web_server.py`、`gateway/platforms/api_server.py`）
- 未覆盖范围：前端/桌面代码；各 IM 适配器内部协议细节

## 证据分类

- Evidence：函数/类/路由定义
- Inference：后端分层职责
- Unknown：生产部署实际启用的 extra/provider

## 核心结论

[Inference] "后端"在 Hermes 中并非传统三层 Web 后端，而是**以 `AIAgent` 运行时为核心、多入口驱动**的 agent 服务。两个 HTTP 服务面：FastAPI Dashboard（`hermes_cli/web_server.py`）与 aiohttp OpenAI 兼容 API（`gateway/platforms/api_server.py`）。

---

## 1. 后端入口 / 初始化流程

| 入口 | 位置 | 初始化 | Evidence |
| --- | --- | --- | --- |
| `hermes`（CLI/TUI） | `hermes_cli/main.py:main:13166` | 配置加载→构造 `AIAgent`（`cli_agent_setup_mixin.py:_init_agent:226`） | 同 |
| `hermes gateway` | `gateway/run.py:main:21755` → `asyncio.run(start_gateway)` | 加载平台适配器、注册 inbound handler、启动 cron tick | `run.py:7314,7330` |
| `hermes serve`/`dashboard` | `hermes_cli/web_server.py`（FastAPI `app:264`，`lifespan`） | 挂载 SPA、注册路由、启动后台 action 子进程 | `subcommands/dashboard.py:101,136` |
| `hermes-acp` | `acp_adapter/server.py:AcpAgent:451` | JSON-RPC over stdio，`run_in_executor` 跑同步 agent | `server.py:1562` |
| `mcp serve` | `mcp_serve.py:run_mcp_server:959` | stdio MCP 服务端 | 同 |

[Evidence] 通用初始化：`hermes_bootstrap.py`（Windows UTF-8 stdio，每个入口最先 import）、`hermes_logging.py:setup_logging:259`、`hermes_cli/config.py:load_config`。

---

## 2. 配置加载 / 日志初始化 / 服务注册

- 配置：`~/.hermes/config.yaml`（`hermes_cli/config.py`，RLock 守护的缓存 + 原子写）；环境变量 `HERMES_*`（`hermes_cli/env_loader.py`）；profile 覆盖（`hermes_cli/profiles.py`）。
- 日志：`hermes_logging.py:setup_logging`（队列 handler、会话上下文过滤、跨进程文件锁）。
- 服务/路由注册：
  - Dashboard FastAPI：`@app.get/post` 装饰器散布于 `web_server.py`。
  - Gateway OpenAI API：路由表 `api_server.py:1480`（list of tuples）。
  - 工具：`registry.register(...)` at import（`tools/*.py`）。
  - 平台：`gateway/platform_registry.py:PlatformRegistry:162` + 插件 `register_platform`。

---

## 3. API / 路由 / Handler 清单（代表性）

### 3.1 Dashboard FastAPI（`hermes_cli/web_server.py`）
| 路由 | 方法 | 处理 | Evidence |
| --- | --- | --- | --- |
| `/api/chat/image-upload` | POST | 聊天图片上传 | `:1878` |
| `/api/files`, `/api/files/read`, `/api/files/download`, `/api/files/upload(-stream)`, `/api/files/mkdir`, DELETE `/api/files` | * | 文件管理（NS-501 多部分上传） | `:1921-2169` |
| `/api/fs/{list,read-text,write-text,read-data-url,git-root,default-cwd}` | GET/POST | 文件系统操作 | `:2169-2296` |
| `/api/git/{status,worktrees,branches,...,review/*,worktree/*,branch/switch}` | * | Git 集成 | 多处 |
| `/api/media` | GET | 媒体解析为 data-url | `:1601` |
| `/api/status` | GET | 状态 | `:2599` |
| `/api/system/stats` | GET | 系统统计 | `:2865` |
| `/api/curator*` | * | 自我改进控制 | `:2955-2988` |
| `/api/learning/{graph,node}` | GET | 学习图 | `:2998,3026` |
| `/api/pty` | WS | PTY 桥（Chat tab 终端） | `:16186` |
| `/api/ws` | WS | JSON-RPC sidecar | `:16363` |
| `/api/console` | WS | 控制台 | `:15836` |

[Evidence] 中间件：CORS、body-limit、security headers、plugin 路由 enabled/disabled 门控（`web_server.py:464,494,570,576,600`）。后台动作经 `subprocess.run` 拉起 `hermes` CLI（`web_server.py:1572, _spawn_hermes_action:2992/3126`）。

### 3.2 Gateway OpenAI 兼容 API（`gateway/platforms/api_server.py`，aiohttp）
| 路由 | 处理 | Evidence |
| --- | --- | --- |
| `POST /v1/chat/completions` | `_handle_chat_completions:2552` | `:1497` |
| `POST /v1/responses` | `_handle_responses:3685` | — |
| `POST /v1/runs`（SSE） | `_handle_runs:4677` | — |
| `GET /v1/models` | — | `:1903` |
| `GET /v1/capabilities` | — | `:1951` |
| `GET /v1/skills` | — | `:2031` |
| `GET /v1/toolsets` | — | `:2062` |
| `/v1/sessions*` | — | `:2187-2323` |
| `/v1/jobs*` | — | `:4038-4246` |
| `POST /api/cron/fire`（可选） | — | `:1522` |
| Per-profile 前缀 `/p/<profile>/v1/...` | — | `:35` |

[Evidence] 鉴权 Bearer（`_check_auth:1234`）；幂等缓存 `_IdempotencyCache:798`；CORS/body-limit/security 中间件（`:584,752,788`）。

### 3.3 一个 API/Handler 的标准描述格式

以 `POST /v1/chat/completions` 为例：
```
入口：api_server.py:_handle_chat_completions:2552（aiohttp handler）
→ 参数：OpenAI chat completions body（messages, model, tools, stream...）
→ 参数验证：_normalize_chat_content / _multimodal_validation_error（:167,373）
→ 鉴权：_check_auth:1234（Bearer）
→ 调用服务：APIServerAdapter → 经 ResponseStore（:404）关联会话 → 构造 AIAgent → run_conversation
→ 数据访问：SessionDB（hermes_state.py）
→ 事务边界：单次请求；幂等键 300s（_IdempotencyCache:798）
→ 错误类型：OpenAI 风格 error JSON（_openai_error:678）
→ 调用方：任意 OpenAI 兼容客户端
→ Evidence：api_server.py:1497,2552; hermes_state.py:971
```

---

## 4. 应用服务 / 领域逻辑 / 数据访问

[Inference] 无经典 service/repository 分层。"应用服务"角色由 `agent/` 与 `gateway/run.py` 的 `_run_agent_inner` 承担；"数据访问"由 `hermes_state.py:SessionDB`（SQLite + FTS5）+ 各 JSON 存储（cron jobs、async_delegations SQLite）承担。

- `SessionDB`（`hermes_state.py:971`）：消息/会话/网关路由/压缩冷却；WAL 调优、写重试带抖动、schema 修复（`repair_state_db_schema:606`）。
- `cron/jobs.py`：`jobs.json` + 跨进程文件锁（fcntl/msvcrt）。
- `tools/async_delegation.py:93`：`async_delegations` SQLite 表。
- `hermes_cli/kanban_db.py`、`gateway/rich_sent_store.py`、`gateway/platforms/api_server.py:ResponseStore:404`：各子系统 SQLite。

---

## 5. 数据模型 / Schema / Migration

[Evidence]
- 会话/消息 schema 由 `SessionDB` 在初始化时建表（含 FTS5 虚拟表，`hermes_state.py:5,11,470`）；运行时 schema 修复（`repair_state_db_schema:606`）。
- 未发现独立 migration 框架/目录；schema 演进以"建表 + ALTER + 修复"方式内联。
- Cron/async_delegation 表同样内联 `CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ADD COLUMN`（`tools/async_delegation.py:114-123`）。

---

## 6. 事务 / 并发 / 缓存 / 队列 / 后台任务

- **并发**：`AIAgent` 同步核心；工具并行 `ThreadPoolExecutor`（`tool_executor.py:327`）；gateway asyncio；ACP `run_in_executor`。
- **锁**：per-agent `threading.RLock`（`run_agent.py:3972`）；cron 跨进程文件锁；SessionDB 写重试。
- **事务边界**：SQLite 默认自动提交粒度；写重试带抖动；async_delegation 状态机（`state` 列：running/finalizing/delivered）。
- **缓存**：工具 `check_fn` TTL 缓存（30s + 60s last-good，`registry.py:150-215`）；技能扫描 TTL（`skills_tool.py:90-105`）；prompt cache（`agent/prompt_caching.py`）。
- **队列/后台任务**：async_delegation daemon `ThreadPoolExecutor`（`async_delegation.py:391`）；cron 调度线程池（`scheduler.py:501,515`）；auxiliary LLM 调用（`agent/auxiliary_client.py`）；background review（`agent/background_review.py`）；dashboard 后台 action 子进程。

---

## 7. 鉴权 / 权限 / 安全边界

- **Dashboard**：本地鉴权令牌/cookie（`hermes_cli/dashboard_auth/`、`dashboard_register.py`、`web_server.py` 中间件）。
- **Gateway OpenAI API**：Bearer token（`api_server.py:_check_auth:1234`）。
- **IM 网关**：DM pairing 授权谁可对话（`gateway/pairing.py`、`authz_mixin.py`）；slash 命令访问控制（`slash_access.py`）。
- **命令审批**：`tools/approval.py`（173KB）+ `write_approval_commands.py`；gateway 审批事件（`api_server.py:71`）。
- **凭据**：`agent/credential_pool.py`、`agent/credential_sources.py`、`agent/secret_sources/`、`hermes_cli/secrets_cli.py`、`hermes_cli/onepassword_secrets_cli.py`。
- **工具 override 策略**：`tools/registry.py:316-365`（`PluginToolOverrideError`）。
- **供应链**：精确 pin、lazy install、MCP 可疑过滤、目录 pin 规则、SSL guard（`agent/ssl_guard.py`、`ssl_verify.py`）、redact（`agent/redact.py`）。

---

## 8. 文件系统访问 / 外部服务 / 错误处理

- 文件系统：`tools/file_tools.py`（read/write/patch/search）、`tools/file_operations.py`；路径解析含容器/会话 cwd override（`_resolve_path`、`_authoritative_workspace_root`）。
- 外部服务：LLM provider、IM SDK、沙箱（`tools/environments/`）、MCP server、浏览器 CDP（`tools/browser_*`）、图像/视频/TTS/STT/web search provider。
- 错误处理：工具错误转 JSON（`registry.dispatch`）；LLM fallback 链（`try_activate_fallback`）；gateway 异常隔离（`_gateway_loop_exception_handler`）；`error_classifier.py`。

---

## 9. 扩展方式 / 测试方式 / 调试入口 / 已知限制

- 扩展：见 `扩展机制.md`（工具 `registry.register`、插件 `register(ctx)`、provider、skill、MCP、cron、ACP）。
- 测试：见 `测试与CI.md`（pytest，per-file 子进程隔离，hermetic conftest）。
- 调试：`hermes doctor`（`hermes_cli/doctor.py`）、`hermes_cli/debug.py`、`agent/stream_diag.py`、`--verbose`、debugpy（dev extra）。
- 已知限制 / 技术债：
  - 巨型文件（`cli.py`、`run_agent.py`、`gateway/run.py`、`agent/auxiliary_client.py`）。
  - `run_agent.py` 半拆分状态。
  - import 时副作用（`cli.py:730`）。
  - 无统一 ORM/migration 框架。

---

## 已确认事实

- 两套 HTTP 服务（FastAPI dashboard + aiohttp gateway API），各自路由与中间件。
- SQLite+FTS5 为主要持久化；无 ORM/migration 框架。
- 多入口共享 `AIAgent` 运行时。

## 合理推断

- "应用服务/数据访问"职责内联于运行时与 gateway，而非独立层。

## Unknown 与待验证事项

- 各 SQLite 子库在生产的数据量级与迁移历史（无 migration 文件可追溯）。

## 批判性评估

- Dashboard 后台动作直接 `subprocess.run` 拉起 `hermes`，存在命令注入与权限边界模糊风险（虽有审批）。
- 多套独立 SQLite/JSON 存储缺乏统一事务/一致性。

## 建设性改善建议

- [Recommendation] 抽取统一的"服务/数据访问"层，使 Dashboard/gateway/ACP 共享而非各自内联。优先级：中；难度：高。
- [Recommendation] 引入显式 schema migration（如 alembic 或轻量自管），取代内联 ALTER。优先级：低；难度：中。

## 主要证据索引

- `hermes_cli/web_server.py:264,1601,1878,1921-2296,2599,2865,2955-3026,15836,16186,16363`
- `gateway/platforms/api_server.py:35,404,798,917,1234,1480,1497,1903,2031,2062,2187,2552,3685,4038,4677`
- `hermes_state.py:971,606`、`cron/jobs.py:71`、`tools/async_delegation.py:93,391`
- `agent/chat_completion_helpers.py:1407`、`tools/registry.py:614`、`tools/approval.py`
- `gateway/run.py:9235,17586,17736`、`gateway/pairing.py`、`gateway/authz_mixin.py`
- `hermes_cli/dashboard_auth/`、`hermes_cli/dashboard_register.py`

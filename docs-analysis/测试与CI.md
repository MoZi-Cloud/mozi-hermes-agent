# 测试与CI.md — 测试体系与 CI/CD

## 分析快照

- 分支：main
- HEAD：a9cc17fd80648bfee0d0b677fa9ea91421f329fc
- 工作区状态：clean
- 子模块状态：无
- 分析时间：2026-07-18
- 分析范围：`pyproject.toml`（pytest/ruff/ty 配置）、`tests/`（2090 test_*.py）、`tests-js/`、`.github/workflows/*.yml`、`scripts/run_tests*.py`、`Dockerfile`、`docker/`、`nix/`、`packaging/`、`upload_to_pypi.yml`
- 未覆盖范围：单个测试用例内容（仅统计与抽样）

## 证据分类

- Evidence：配置文件、workflow 文件、目录统计
- Inference：CI 实际执行边界
- Unknown：CI 在 fork 上是否完整运行（多处 `github.repository == 'NousResearch/hermes-agent'` 门控）

## 核心结论

[Evidence] 测试框架为 **pytest 9.0.2 + pytest-asyncio 1.3.0 + pytest-timeout**（`pyproject.toml:160` dev extra）。测试以**每文件单子进程**方式运行（`scripts/run_tests_parallel.py`，非 xdist），hermetic 由 `tests/conftest.py` 的 autouse fixture 强制保证。CI 由单一编排器 `ci.yml` 扇出多个子 workflow，`all-checks-pass` 聚合门。

[Evidence] **集成测试默认被排除**（`pyproject.toml` `addopts = "-m 'not integration'"`），仅专用 e2e job 用 `-m integration` 运行。

---

## 1. 测试框架与配置

- 框架：pytest 9.0.2、pytest-asyncio 1.3.0（316 文件用 `@pytest.mark.asyncio`）、pytest-timeout（`pyproject.toml:160`）。
- pytest 配置（`pyproject.toml:359` `[tool.pytest.ini_options]`）：
  - `testpaths = ["tests"]`。
  - markers：`integration`（外部服务）、`real_concurrent_gate`（opt-out autouse stub）。
  - `addopts = "-m 'not integration'"`（默认全量排除集成测试）。
- **无覆盖率门**：无 `[tool.coverage]`、dev extra 无 coverage、CI 无 `--cov`。
- ty（类型检查，`pyproject.toml:368`）：`python-version=3.13`、`unknown-argument=warn`、`redundant-cast=ignore`——**仅建议，CI 不阻断**。
- ruff（`pyproject.toml:375`）：preview 模式，**仅启用 `PLW1514`（unspecified-encoding）一条规则**；其余 lint 故意关闭。`tests/**`、`skills/**`、`optional-skills/**`、`plugins/**` per-file ignore（`:388-394`）。
- 无 xdist；并发靠 per-file 子进程（`scripts/run_tests_parallel.py`）。
- hermetic fixture（`tests/conftest.py`，897 行，均 autouse）：
  - `_hermetic_environment`（`:328`）：清空凭据型 env（`:57-159`）与行为型 `HERMES_*`（`:171-325`）、重定向 `HERMES_HOME` 到 per-test tempdir、`TZ=UTC`、`LANG=C.UTF-8`、`PYTHONHASHSEED=0`、禁 AWS IMDS + Tirith。
  - `_live_system_guard`（`:547`）：包装 `os.kill`/`os.killpg`/`subprocess.*`/`os.system`/`os.popen`/`pty.spawn`/`asyncio.create_subprocess_*`，拒绝触碰真实 `hermes-gateway`；可经 `@pytest.mark.live_system_guard_bypass` 绕过。
  - `_ensure_current_event_loop`（`:453`）：给同步测试提供事件循环。
- skip 统计：86 `@pytest.mark.skip`、24 `skipif`、~181 xfail；仅 1 文件用 `@pytest.mark.integration`。

---

## 2. 测试布局与统计（2090 test_*.py）

| 子目录 | 文件数 | 子目录 | 文件数 |
| --- | --- | --- | --- |
| `tests/gateway/` | 492 | `tests/tools/` | 323 |
| `tests/hermes_cli/` | 443 | `tests/run_agent/` | 143 |
| `tests/agent/` | 261 | `tests/plugins/` | 84 |
| `tests/cli/` | 90 | `tests/cron/` | 35 |
| `tests/tui_gateway/` | 33 | `tests/docker/` | 26 |
| `tests/acp/` | 15 | `tests/skills/` | 14 |
| `tests/honcho_plugin/` | 10 | `tests/stress/` | 10 |
| `tests/integration/` | 9 | `tests/hermes_state/` | 8 |
| `tests/providers/` | 7 | `tests/e2e/` | 5 |
| `tests/computer_use/` | 4 | 其他（website/acp_adapter/secret_sources/dashboard/manual/fakes/fixtures/scripts/ci 等） | 1–3 |

分层子目录：
- `tests/integration/`（9）：需外部服务（Modal/Daytona/vision docker/voice/batch resume），默认排除。
- `tests/e2e/`（5）：discord/platform/matrix_xsign/wheel_locales；专用 e2e CI job 运行。
- `tests/stress/`（10）：并发、属性 fuzz、基准、子进程 e2e。
- `tests/docker/`（26）：对构建镜像运行（`HERMES_TEST_IMAGE`，`docker.yml` 驱动）。

实际被覆盖（抽样）：
- agent loop / run_agent：`tests/run_agent/`（143，含回归用例如 `test_1630_context_overflow_loop.py`、`test_28161_anthropic_stream_pool_cleanup.py`）。
- gateway：`tests/gateway/`（492，平台适配器、auto-reset、relay、restart）。
- FastAPI/dashboard：51 文件用 `TestClient`（`test_dashboard_*`、`test_web_server_*`、`test_mcp_dashboard_oauth`）。
- TUI：`ui-tui/src/__tests__/`（TS vitest：platform、bundleNoAsyncEsmDeadlock）；Python 侧 `tests/tui_gateway/`（33）+ `test_pty_session.py` 等。
- JS 单测：`tests-js/`（3：assistant-ui-tap-compat、desktop-mac-entitlements、package-json-lazy-deps）。

JS 工具：npm workspaces（`apps/*`、`ui-tui`、`ui-tui/packages/*`、`web`、`tests-js`），Node 22，共享 ESLint `eslint.config.shared.mjs`，Prettier `.prettierrc`（no semi/single/120）。每 workspace `check`=typecheck+test，`fix`=eslint --fix。

---

## 3. CI workflow 表

编排器 `.github/workflows/ci.yml`（`pull_request` + `push:main`）：`detect` 作业（`scripts/ci/classify_changes.py` 经 `.github/actions/detect-changes`）→ 扇出 `workflow_call` 子 workflow → `all-checks-pass` 聚合（`ci.yml:146-181`；docker **故意不**作为必需，`ci.yml:160`）。

| workflow | 触发 | 主要 job | 跑测试? | OS 矩阵 |
| --- | --- | --- | --- | --- |
| `ci.yml` | PR, push:main | detect/扇出/all-checks-pass/ci-timings | 编排 | ubuntu |
| `tests.yml` | workflow_call | generate（LPT 分片，`test_durations.json`）/ test（8 分片矩阵，fail-fast:false，30m 超时）/ save-durations（main）/ e2e | **是**（`scripts/run_tests.sh --files`）；e2e 跑 `tests/e2e/` + wheel locale `-m integration` | ubuntu, Py3.11, `uv sync --locked --extra all --extra dev` |
| `lint.yml` | workflow_call | lint-diff（建议 ruff+ty，不阻断）/ ruff-blocking（`ruff check .`）/ windows-footguns（`scripts/check-windows-footguns.py`）/ ci-review（CI 敏感文件需 `ci-reviewed` 标签） | 否 | ubuntu |
| `js-tests.yml` | workflow_call | workspaces 矩阵 + check（`npm run check`+`fix`，fail-fast:false） | 是（JS/TS） | ubuntu, Node 22 |
| `docker.yml` | release:published / workflow_call | build（amd64+arm64，`:test`，跑 `tests/docker/`）/ merge（manifest list） | **是**（`scripts/run_tests.sh tests/docker/`） | amd64 + arm64 |
| `docker-lint.yml` | workflow_call | hadolint（Dockerfile，warning）/ shellcheck（`docker/`，error） | 否 | ubuntu |
| `supply-chain-audit.yml` | workflow_call（PR） | scan（.pth/base64+exec/subprocess 编码/install-hook 模式，命中失败）/ dep-bounds（无界 `>=` 依赖失败）/ mcp-catalog-review（需 `mcp-catalog-reviewed` 标签） | 否 | ubuntu |
| `osv-scanner.yml` | workflow_call + 周一 09:00 cron + workflow_dispatch | scan（uv.lock/package-lock.json/website，`fail-on-vuln:false` 仅检测） | 否 | reusable |
| `contributor-check.yml` | workflow_call | check-attribution（新作者邮箱须在 `scripts/release.py AUTHOR_MAP`） | 否 | ubuntu |
| `history-check.yml` | workflow_call（PR） | check-common-ancestor（拒绝无关历史 PR） | 否 | ubuntu |
| `uv-lockfile-check.yml` | workflow_call | `uv lock --check` | 否 | ubuntu |
| `lockfile-diff.yml` | workflow_call（npm 变更） | advisory 语义 diff 评论（不阻断） | 否 | ubuntu |
| `docs-site-checks.yml` | workflow_call | extract-skills/generate-skill-docs/lint:diagrams/Docusaurus build | 否 | ubuntu, Node22+Py3.11 |
| `js-autofix.yml` | push:main（JS/TS 变更）+ dispatch | generate-patch（无特权 eslint --fix）/ apply-patch（特权 bot PR + auto-merge） | 否 | ubuntu |
| `deploy-site.yml` | release/push:main/dispatch | deploy-vercel / deploy-docs（Pages） | 否 | ubuntu |
| `skills-index.yml` | 每日 2 次 + dispatch + push:main | build-index（`scripts/build_skills_index.py`）/ trigger-deploy | 否 | ubuntu |
| `skills-index-freshness.yml` | 每 4h + dispatch | check-freshness（>26h 陈旧则开 issue） | 否 | ubuntu |
| `upload_to_pypi.yml` | push: tags `v20*`（CalVer）+ dispatch | build（web+TUI 构建入 `hermes_cli/{web,tui}_dist`，`uv build`）/ publish（OIDC trusted publishing）/ sign（Sigstore 附 Release） | 否 | ubuntu |

其他 `.github/`：`dependabot.yml`、`actions/{detect-changes,nix-setup,retry}`、`ISSUE_TEMPLATE/*`、`PULL_REQUEST_TEMPLATE.md`。

---

## 4. CI 中实际强制（阻断）的质量门

`all-checks-pass` 门内（`ci.yml:146-181`；仅 `failure` 失败，`skipped` 通过）：
- Python 测试（8 分片，`tests.yml`）。
- ruff `check .`（仅 `PLW1514`，`lint.yml:118`）。
- Windows footguns 静态扫描（`lint.yml:146`）。
- JS/TS typecheck + test（`js-tests.yml`）。
- docker-lint（hadolint + shellcheck）。
- `uv.lock` sync 检查。
- history-check、contributor-check（PR）。
- supply-chain scan + dep-bounds + mcp-catalog-review（PR 相关路径）。
- ci-review 标签门（CI 敏感文件变更）。
- e2e job（`tests.yml:156`）。

仅建议（不阻断）：lint-diff（ruff+ty delta）、lockfile-diff、osv-scanner（`fail-on-vuln:false`）、ci-timings。**ty 类型检查全程仅建议**。

**明确不**必需：docker 构建（`ci.yml:160` 显式排除）；**无覆盖率门**。

---

## 5. 测试覆盖矩阵（按模块抽样）

| 模块/功能 | 单元 | 集成 | E2E | CI 执行 | 主要缺口 |
| --- | --- | --- | --- | --- | --- |
| agent loop / run_agent | ✓（`tests/run_agent/` 143） | 部分（integration） | — | 单元✓ / 集成仅 e2e job | 真实 provider 端到端靠 integration |
| gateway / 平台适配器 | ✓（`tests/gateway/` 492） | 部分 | discord/platform（`tests/e2e/`） | 单元✓ | 真实 IM 平台联调（凭据依赖） |
| tools | ✓（`tests/tools/` 323） | sandbox 集成 | — | ✓ | 沙箱后端真实环境覆盖 |
| dashboard FastAPI | ✓（TestClient 51 文件） | — | — | ✓ | 浏览器端到端 |
| TUI | ✓（vitest + `tests/tui_gateway/` 33） | — | — | ✓ | 真实终端渲染 |
| cron | ✓（`tests/cron/` 35） | — | — | ✓ | 长周期调度真实运行 |
| plugins | ✓（`tests/plugins/` 84） | — | — | ✓ | 第三方 entry-point 插件 |
| skills | ✓（`tests/skills/` 14） | — | — | ✓ | Skills Hub 真实市场 |
| ACP | ✓（`tests/acp/` 15 + `acp_adapter/` 3） | — | — | ✓ | 真实编辑器客户端 |
| Docker | — | ✓（`tests/docker/` 26 对镜像） | — | 仅 docker.yml（非必需门） | — |
| 性能/压力 | ✓（`tests/stress/` 10） | — | — | ✓ | 无持续基准阈值 |

---

## 6. "存在/配置/CI 执行"四层区分（关键）

[Evidence] 本仓库严格区分以下层次，避免误判：
- **存在测试文件**：`tests/` 2090 文件。
- **测试命令被配置**：`scripts/run_tests.sh`、各 workspace `npm run check`。
- **CI 执行**：`tests.yml` 8 分片 + e2e；`js-tests.yml`；`docker.yml` 跑 `tests/docker/`（但 docker 非必需门）。
- **被跳过/条件禁用**：`integration` 默认排除（`addopts`）；86 skip + 24 skipif + ~181 xfail；`live_system_guard_bypass` 标记。
- **CI 配置存在但可能失效**：docker 构建/测试在 fork 上因 `github.repository` 门控可能 no-op；`upload_to_pypi` 同理。
- **README 声称但无源码证据**：未发现（README 主要声明均有源码/测试支撑）。

---

## 7. Docker / 部署 / 发布

- **Dockerfile**（~20KB，多阶段）：`uv_source`（`ghcr.io/astral-sh/uv:0.11.6-python3.13-trixie` SHA pin）、`node_source`（`node:22-bookworm-slim` SHA pin）、运行时基 `debian:13.4`；apt 装 curl/ripgrep/ffmpeg/gcc/cmake/libolm-dev/git/docker-cli；**s6-overlay v3.2.3.0**（SHA per arch）监管，`/init` 为 PID 1；非 root `hermes` UID 10000（`/opt/data`）；`uv sync` 入 `/opt/hermes/.venv`；Playwright 浏览器 `/opt/hermes/.playwright`；`HERMES_LAZY_INSTALL_TARGET=/opt/data/lazy-packages`；`VOLUME ["/opt/data"]`；`ENTRYPOINT ["/init","/opt/hermes/docker/main-wrapper.sh"]`。
- **docker-compose.yml**：两服务 `gateway`（`command: gateway run`）+ `dashboard`（`command: dashboard --host 127.0.0.1 --no-open`），均 `network_mode: host`，挂载 `~/.hermes:/opt/data`。
- **docker.yml workflow**：amd64+arm64 构建 `:test`→跑 `tests/docker/`→按 digest 推送→manifest list（`:main`/`:latest`/release tag）。门控 `github.repository == 'NousResearch/hermes-agent'`。
- **PyPI 发布**（`upload_to_pypi.yml`）：CalVer tag `v20*`（`scripts/release.py`）→ build（含前端构建入 `web_dist`/`tui_dist`）→ OIDC trusted publishing（`skip-existing`）→ Sigstore 签名附 Release。
- **Homebrew**：`packaging/homebrew/hermes-agent.rb`。
- **Nix**：`flake.nix` + `nix/*.nix`（flake-parts + uv2nix + pyproject.nix；x86_64-linux/aarch64-linux/aarch64-darwin）。
- **安装器**：`setup-hermes.sh`、`scripts/install.{sh,ps1,cmd}`（入 wheel `hermes_cli/scripts/`）。
- **setup.py**：定制只读源码树 build/egg_info；`skills/`、`optional-skills/` 作 data_files。

---

## 8. 规范测试运行器（契约）

`scripts/run_tests.sh`（本地==CI）：
- 定位 venv（`.venv`/`venv`/`~/.hermes/.../venv`/Nix `HERMES_PYTHON`）→ `compileall` 预编译 `git ls-files '*.py'` → `exec env -i PATH HOME TZ=UTC LANG=C.UTF-8 LC_ALL PYTHONHASHSEED=0 … python scripts/run_tests_parallel.py "$@"`。
- `scripts/run_tests_parallel.py`：发现测试文件（排除 integration/e2e/docker，`_SKIP_PARTS`~`:63`），**每文件一个 `python -m pytest <file>` 子进程**，有界并行（默认 `os.cpu_count()`），每文件 300s 超时（`--file-timeout`）。`--generate-slices N` 用 `test_durations.json` 生成 LPT 均衡矩阵。

---

## 已确认事实

- pytest + per-file 子进程隔离 + 强 hermetic conftest；集成测试默认排除。
- CI 编排器扇出多子 workflow；阻断门不含 docker、不含覆盖率。
- 发布经 OIDC + Sigstore；docker 多架构 + s6-overlay。

## 合理推断

- per-file 子进程模型牺牲启动开销换取隔离与稳定性，适合大量 import 时副作用重的代码库。

## Unknown 与待验证事项

- CI 在 fork 上的实际覆盖（repository 门控路径）。
- 真实测试通过率与 flakes（未动态运行）。
- 覆盖率数值（未配置收集）。

## 批判性评估

- **无覆盖率门**——无法量化测试盲区。
- ty（类型检查）仅建议不阻断，类型回归可能漏入。
- 集成测试默认排除，真实 provider/IM/沙箱联调覆盖有限。
- ruff 仅启用单条规则，lint 覆盖极弱（依赖 review 而非自动门）。
- docker 测试非必需门，容器回归可能滞后发现。

## 建设性改善建议

- [Recommendation] 引入覆盖率收集与非阻断趋势报告，逐步建立基线。优先级：中；难度：低。
- [Recommendation] 将 ty 类型检查从建议升级为阻断（或至少对 `agent/`、`gateway/` 核心目录阻断）。优先级：中；难度：中。
- [Recommendation] 将 docker 测试纳入 `all-checks-pass` 必需门（或至少在 release 前阻断）。优先级：低；难度：中。
- [Recommendation] 扩展 ruff 规则集（逐步启用），减少对人工 review 的依赖。优先级：低；难度：低。

## 主要证据索引

- `pyproject.toml:160(dev extra),359(pytest),368(ty),375(ruff),388-394,307-310`
- `.github/workflows/ci.yml:29,146-181(160 docker 排除)`
- `.github/workflows/tests.yml:41,156`、`lint.yml:118,146,166`、`docker.yml:24`、`upload_to_pypi.yml`
- `scripts/run_tests.sh`、`scripts/run_tests_parallel.py:~63(_SKIP_PARTS)`
- `tests/conftest.py:328,453,547`
- `Dockerfile`、`docker-compose.yml`、`flake.nix`、`packaging/homebrew/hermes-agent.rb`、`scripts/release.py`

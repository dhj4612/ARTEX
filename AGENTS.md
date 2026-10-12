# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## 项目概览

ARTEX 是 LLM 多 agent 驱动的自主渗透测试系统：**Go 单体后端**（`net/http`，前端静态产物经 `go:embed` 内嵌进单二进制）+ **Next.js 前端** + **PostgreSQL**。agent 能力由 `github.com/Autumn-27/norma` SDK 提供（`agentcore` / `tool` / `permission` / `harness` / `memory` / `transcript`）。

核心是**双图架构**（README「系统技术架构」一节有完整 mermaid 图，改动核心链路前先读）：
- **资产图**（全局共享，`assets` / `companies` / `company_scope` / `task_scope`）：跨任务的资产真值库，域名→子域→服务→端点的父子关系与去重 key 全部由程序计算，agent 只提交原始信息。
- **探索图**（每任务独立，`exploration_nodes` / `exploration_edges` / `activity` / `exploration_anchors`）：一次任务的推进链，节点为 `goal / intent / fact / finding / hint`，靠 `spawns / derived_from / yields / proves` 连成血缘链。
- 两图靠 `exploration_anchors(node_id, asset_id)` 相连，支撑资产测试覆盖度与覆盖图。

## 常用命令

```bash
# 开发（后端 :8787 + 流量代理 :8788 + 前端 next dev :5173，Ctrl-C 一并退出）
./dev.sh

# 后端（不内嵌前端 → 内嵌的 webui handler 直接 404，需另跑 next dev）
go run ./cmd/artex -addr :8787 -proxy 127.0.0.1:8788

# 前端热更新（/api 反代到 :8787，可用 AUTOPENTEST_API 覆盖后端地址）
cd web && npm run dev
cd web && NEXT_PUBLIC_MOCK=1 npm run dev   # 纯 mock 预览，无需后端

# 测试（需要一个可连的 PostgreSQL，见下）
go test ./...
go test ./db -run TestAssetsInsert -v      # 单测试
go test ./server -run TestEngineCancelCause -v

# 前端 lint / 格式化（Biome）
cd web && npm run check        # 检查
cd web && npm run format       # 格式化
cd web && npm run check:fix    # 自动修

# 单二进制构建（-tags embedui 才内嵌前端）
cd web && npm run build:static && cd ..
cp -r web/out server/webui/dist
CGO_ENABLED=0 go build -tags embedui -o artex ./cmd/artex

# 跨平台 Release 打包（内含前端构建，产物 dist/artex-<ver>-*.zip）
./build.sh --release
ARTEX_TARGETS=linux/amd64,windows/amd64 ./build.sh --release
```

### 测试对 PostgreSQL 的依赖（重要）

`db` / `agent` / `server` / `evidence` 四包的 `TestMain` 会读取 `config.json` 的 `database` 或环境变量 `ARTEX_PG_DSN` 连接 Postgres，并**拿同一个 advisory lock（`7337741002`）串行化整个测试套件**——目的是避免各包 `go test ./...` 并行时互相 `DELETE` 清理数据。无 DB 配置时相关测试会自行 skip，不会失败。

数据库配置解析优先级见 `config/config.go`：`ARTEX_PG_DSN` > `config.json` 的 `database.dsn` > 分字段组装。

前端有若干 `web/src/lib/*.test.mjs`，用 `node --test` 直接可跑（未接入 npm script / CI）。

## 代码地图

| 目录 | 职责 |
| --- | --- |
| `cmd/artex/main.go` | 入口：解析 `-addr` / `-data` / `-proxy` → `selfupdate.Bootstrap()` → `server.NewManager` → `server.New` → `http.Server`。退出码是更新协议的一部分（`ExitRestart=75` 触发守护脚本重启换装） |
| `server/` | `net/http` 路由 + JWT 鉴权 + SSE；`Manager`（任务/引擎/store 生命周期）、`Engine`（事件驱动调度循环）、`Orchestration`（多任务编排）、`scheduler`、`notifier` |
| `agent/` | 无预编译模拟，必须要有 LLM provider。`planner`（唯一意图生成者）、`worker`（执行者 ×N）、`goals`、`mainagent`（人在环路）、`tools.go`（把双图暴露成 LLM 工具，最大的文件）、`provider`、`compaction`、`wrapup` |
| `db/` | 双图的 Postgres 落地（pgx）。`schema.sql`（78K，唯一 schema 来源）、`exploration.go`（76K，探索图读写）、`assets.go`、`tasks.go`、`config.go`（设置/LLM profile/内置 agent seed） |
| `traffic/` | 记录型 MITM 代理（`go-mitmproxy` + 自签 CA），全程留痕；亦有归档/取证 |
| `guard/`、`intercept/` | 工具审批门：规则命中 + 模型兜底审批（judge），人在环路审批 |
| `llmpool/`、`llmrec/` | LLM 多 profile 路由/健康/故障转移；LLM 请求记录与 token 计量 |
| `enrich/` | 资产 DNS / HTTP 异步补全 |
| `mcphttp/` | MCP 客户端（Streamable HTTP / 旧版 SSE） |
| `notify/` | 漏洞 IM 推送渠道（钉钉/飞书/企微/Webhook/Telegram/邮件） |
| `report/`、`evidence/`、`selfupdate/` | 报告导出、证据快照、页面一键更新（下载→校验 SHA256→冒烟→暂存→换装→失败回滚） |
| `sidequestion/` | `/btw` 旁路提问子系统，有独立设计文档（见下） |
| `skills/` | 内置 skill（每个子目录一个 `SKILL.md`），运行时目录由 `ARTEX_SKILL_DIR` 覆盖 |
| `web/` | Next.js 前端（App Router，`src/app/(main)/...`），静态导出 → `go:embed` |

## 关键约定与坑

- **加表/改列只改 `db/schema.sql`**：`artex` 每次启动幂等重跑整个 schema（`ADD COLUMN` / `CREATE INDEX IF NOT EXISTS`），即「重启即迁移」，没有独立的迁移文件或版本号。升级只换程序、不动数据。
- **新增后端 API**：在 `server/server.go` 的 `Handler()` 里 `mux.HandleFunc("METHOD /api/...")` 注册。`/api/*` 统一走 CORS + `requireAuth`；`auth.go` 只放行 `/api/auth/*` 和 `/api/health`，其余默认要求 JWT（所以「能改程序自身」的接口天然需要登录）。前端在 `web/src/lib/api.ts` 加对应方法，类型放 `web/src/lib/types.ts`。
- **`docs/` 被 gitignore**：代码注释里引用的 `docs §4.3/§4.4`、`docs/任务级超时与收尾设计.md` 等内部设计文档**不在仓库里**，别去找；以代码注释和 `sidequestion/*.md` 为准。
- **`web/src/lib/preferences/theme.ts` 是生成文件**，由 `npm run generate:presets` 产出，pre-commit hook 会重新生成并 `git add`。改主题预设要改生成脚本 `web/src/scripts/generate-theme-presets.ts`，别手改产物。
- **启动必须走 `start.sh` / `start.bat`**（守护脚本）：它按退出码决定是否重新拉起，页面「一键更新」靠它完成换装；直接跑 `./artex` 更新后不会被拉起。版本号为 `dev` 时禁用自更新（避免覆盖本地调试二进制）。
- **`server/webui/dist/` 被 gitignore**：不带 `-tags embedui` 构建时前端不内嵌，`webuiHandler` 直接 404（`server/webui_stub.go`）。
- **Docker 镜像只装工具、不编译**：`Dockerfile` 期望构建上下文里已有 `dist/<TARGETARCH>/artex` 预编译二进制；`./data` 与 `./skills` 以绑定挂载持久化。镜像内 `docker compose up -d` 重建容器会退回镜像自带版本，程序内更新不换镜像。
- **前端有多处 mock 分支**：`web/src/lib/mock/`，由 `NEXT_PUBLIC_MOCK=1` 或 `lib/mock/enabled.ts` 的 `MOCK` 控制，`api.ts` 里两条分支并存，改接口时两边都要看。
- **数据库多表查询优先用 SQL join**，不要在 Go 内存里组装数据。
- **反向代理部署**：SSE 是长连接，反代必须关缓冲（`proxy_buffering off`）否则浏览器收不到事件；实时活动流默认同源，无需配 `NEXT_PUBLIC_SSE_BASE`。

## 参考文档

- `README.md` — 安装/更新/配置/部署说明，以及**系统技术架构**（双图、引擎闭环、worker 过程级信息交换、planner 共享 todolist，含 mermaid 图）。
- `CHANGELOG.md` — 版本历史与各功能的设计取舍说明（含安全审计结论），体积大，按需检索。
- `sidequestion/README.md`、`sidequestion/CONTEXT_BUDGET.md`、`sidequestion/VALIDATION.md` — `/btw` 旁路提问子系统的实现边界、持久化与 HTTP 契约。
- `docs/漏洞流量证据.md` — 漏洞流量证据（`docs/` 未纳入 git，此文件是例外提交的）。

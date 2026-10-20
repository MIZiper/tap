# TAP Analysis Platform

TAP 是一个“多应用协同开发”仓库：每个应用都可以独立运行、独立构建，同时通过明确的库、服务 API 和前端包边界复用相似能力。

当前仓库通过 Git submodule 聚合以下项目：

| 子模块 | Python 包 / 前端包 | 技术栈 | 作用 | 当前复用关系 |
|---|---|---|---|---|
| `dac_core` | `miz-dac` | Python | DAC（Data Action Context）分析框架核心，支持桌面端与 Web 端 | 被 `dac_web` 使用 |
| `dac_web` | `miz-dac_web` / `@mizip/dac_web` | FastAPI + Svelte 5 | DAC 网页版：分析会话、项目存储、可视化、Keycloak | 依赖 `dac_core` |
| `rdb` | `miz-metahub` / `@mizip/metahub` | FastAPI + Svelte 5 | MetaHub：标签、分类、下游节点登记 | 被 `tc` 调用 |
| `rdb` | `miz-tc` / `@mizip/tc` | FastAPI + Svelte 5 | TC（TemplateClient）：通用类型化资源存储、检索、聚合、文件、权限、审阅 | 可作为下游应用的数据层 |
| `tpl` | `tpl-backend` / `tpl-frontend` | FastAPI + Svelte 5 | 测试计划与执行记录 | 当前独立 |
| `tpl` | `rsp-backend` / `rsp-frontend` | FastAPI + Svelte 5 | 风险 / 方案 / 项目 FMEA 管理 | 当前暂缓开发 |

> `rsp` 目前没有开发计划，文档中仅保留端口与目录信息，不作为当前工作重点。

## 现状与目标

当前真实依赖：

```text
dac_web  ──> dac_core

tc       ──> metahub
```

当前尚未发生、但需要逐步靠近的目标：

```text
dac_web ──┐
          ├──> tc（资源存储、搜索、聚合、文件、权限、审阅）
tpl     ──┘

所有应用 ──> Keycloak（统一用户、角色、权限）
所有需要审阅的应用 ──> 统一或可适配的状态流
```

重点目标：

- `dac_web` 和 `tpl` 逐步成为 TC 的实例，复用资源搜索与存储结构。
- 用户系统统一使用 Keycloak，后端验证 JWT，前端只做展示与交互。
- 审阅流程、角色命名、发布状态尽量复用 TC 的机制，避免各应用重复造一套。
- 子模块仍能独立开发、独立测试、独立部署。

## Workspace 结构

根目录同时维护 Python 与前端两个 workspace。

### Python（uv）

根 `pyproject.toml` 的成员：

```text
dac_core
dac_web/backend
rdb/mh-backend
rdb/tc-backend
```

不在根 uv workspace 内：

```text
tpl/tpl-backend
tpl/rsp-backend
```

这两个项目各自维护 `uv.lock`，需要在各自目录下执行 `uv run`。

### 前端（pnpm）

根 `pnpm-workspace.yaml` 的成员：

```text
dac_web/dac-frontend   @mizip/dac_web
rdb/mh-frontend        @mizip/metahub
rdb/tc-frontend        @mizip/tc
tpl/tpl-frontend       tpl-frontend
```

不在根 pnpm workspace 内：

```text
tpl/rsp-frontend
```

根 `overrides` 已为 `@mizip/dac_web`、`@mizip/metahub`、`@mizip/tc` 预置了 `workspace:*`，但当前只有 `@mizip/tc` 实际依赖 `@mizip/metahub`。

## 固定开发端口

固定端口来自各子模块的 `.vscode/tasks.json`，用于本地开发与联调。不要随意改动；如确需修改，必须同步更新对应子模块的 VS Code 任务和根文档。

| 应用 | PostgreSQL | 后端 | 前端 | Keycloak client / realm | VS Code 组合任务 |
|---|---:|---:|---:|---|---|
| `dac_web` | 5431 | 8031 | 5131 | `dac-frontend` / `miz` | DAC |
| `rdb` TC | 5430 | 8030 | 5130 | `miz` / `master`（当前任务配置） | TClient |
| `rdb` MetaHub | 5433 | 8033 | 5133 | 无 | MetaHub |
| `tpl` | 5437 | 8037 | 5137 | `tpl-frontend` / `miz` | TPL |
| `tpl` RSP（暂缓） | 5439 | 8039 | 5139 | 无 | RSP |
| Keycloak | - | 8080 | - | realm `miz` | 外部服务 |

说明：

- 前端通过 `VITE_API_PORT` 指向对应后端；Vite 默认只代理 `/api`（`dac_web` 还代理 `/app`、`/mpl`）。
- `dac_core` 是库，没有固定端口。
- TC 的 Keycloak realm 在当前任务里是 `master`，与 `dac_web`、`tpl` 的 `miz` 不一致。统一认证前不要假设两边角色和 realm 等价。
- `tpl` 的 Keycloak 配置目前只是在任务中预留，后端尚未实现认证。
- MetaHub 与 TC 是两个独立后端，可以分别启动，也可以使用 `rdb` 的 Full MHC 组合任务。

## 快速开始

首次拉取：

```bash
git submodule update --init --recursive
```

安装根 workspace 依赖：

```bash
uv sync
pnpm install
```

注意：

- 根 `uv sync` 只安装根 uv workspace 成员；`tpl/tpl-backend`、`tpl/rsp-backend` 需要进入各自目录运行 `uv run` 时再安装。
- `tpl/rsp-frontend` 不在根 pnpm workspace，需在 `tpl/rsp-frontend` 内单独安装。
- 本地数据库或联调前，如环境中有代理，先取消：
  ```bash
  unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
  ```

## 运行各应用

推荐直接使用各子模块根目录的 VS Code 任务；它们已经配置好固定端口、数据库端口和环境变量。

### dac_core

```bash
# 运行核心测试
uv run pytest dac_core/tests

# 桌面 GUI（需要安装 GUI 可选依赖，见 dac_core/README.md）
uv run dac-bin
```

### dac_web

在 `dac_web/` 下使用 VS Code 任务：

- `(dev) Start DAC PostgreSQL`
- `(dev) Start DAC backend`
- `(dev) Start DAC frontend`
- 或组合任务 `DAC`

手动等价命令（需在 `dac_web/` 下执行，并使用任务中的后端环境变量）：

```bash
cd backend
POSTGRES_PORT=5431 DBSTORE=true \
LOG_DIR=../storage/logs/ PROJECT_DIR=../storage/projects/ \
KEYCLOAK_URL=http://localhost:8080 KEYCLOAK_REALM=miz KEYCLOAK_CLIENT_ID=dac-frontend \
uv run uvicorn --reload --port=8031 dac_web.main:app

cd ../dac-frontend
VITE_API_PORT=8031 pnpm run dev --port 5131
```

### rdb / MetaHub 与 TC

在 `rdb/` 下使用 VS Code 任务：

- MetaHub：`(MetaHub) PostgreSQL`、`(MetaHub) backend`、`(MetaHub) frontend`，或组合任务 `MetaHub`
- TC：`(TClient) PostgreSQL`、`(TClient) backend`、`(TClient) frontend`，或组合任务 `TClient`
- 两者一起：`Full MHC`

手动等价命令：

```bash
# TC
cd rdb/tc-backend
POSTGRES_PORT=5430 \
KC_SERVER_URL=http://localhost:8080 KC_REALM=master KC_CLIENT_ID=miz \
uv run uvicorn --reload --port=8030 tc.main:app

cd rdb/tc-frontend
VITE_API_PORT=8030 pnpm run dev --port 5130

# MetaHub
cd rdb/mh-backend
POSTGRES_PORT=5433 uv run uvicorn --reload --port=8033 metahub.main:app

cd rdb/mh-frontend
VITE_API_PORT=8033 pnpm run dev --port 5133
```

TC 通过 `TC_METAHUB` 访问 MetaHub，默认值为 `localhost:8033`。

### tpl

在 `tpl/` 下使用 VS Code 任务：

- `(TPL) PostgreSQL`
- `(TPL) backend`
- `(TPL) frontend`
- 或组合任务 `TPL`

手动等价命令：

```bash
cd tpl/tpl-backend
DATABASE_URL=postgresql://postgres:password@localhost:5437/tpl \
uv run uvicorn --reload --port=8037 tpl.main:app

cd tpl/tpl-frontend
VITE_API_PORT=8037 pnpm run dev --port 5137
```

### rsp（暂缓）

端口保留为 PostgreSQL `5439`、后端 `8039`、前端 `5139`。当前不安排开发。

## 共享配置速查

| 项目 | 约定 |
|---|---|
| 数据库 | 每个应用独立数据库：`dac`、`tc`、`metahub`、`tpl`、`rsp` |
| TC schema | 启动时自动执行幂等 `tc/db/schema.sql`，并 seed 数据类型 |
| dac_web schema | `backend/dac_web/db/schema.sql`，使用 PostgreSQL 18 的 `uuidv7()` |
| Keycloak | 后端校验 JWT；前端使用 `keycloak-js`，前端角色判断只用于 UI |
| TC 角色 | `nodes:create`、`nodes:read_all`、`nodes:edit_any`、`nodes:review`、`nodes:visibility:{name}` |
| dac_web 管理角色 | `dac-admin`（由 `admin_role` 配置） |
| 敏感字段 | `creator_signature`、client secret、admin token 不得返回前端或写入日志 |
| 代理 | `dac_web` 前端使用 `/api`、`/app`、`/mpl`；其他前端主要使用 `/api` |

## 常用验证命令

```bash
# Python 入口可导入
uv run python -c "import dac_web.main"
uv run python -c "import tc.main"
uv run python -c "import metahub.main"

cd tpl/tpl-backend && uv run python -c "from tpl.main import app"
cd tpl/rsp-backend && uv run python -c "from rsp_backend.main import app"

# 前端类型检查 / 构建（根 workspace 成员）
pnpm --filter @mizip/dac_web check
pnpm --filter @mizip/tc check
pnpm --filter @mizip/metahub check
pnpm --filter tpl-frontend check

# 不在根 workspace 的 RSP 前端
cd tpl/rsp-frontend && pnpm check

# TPL + RSP 集成测试（需先启动两个后端）
python3 tpl/scripts/test-integration.py
```

## 文档索引

- 根开发约定与复用规则：[`AGENTS.md`](./AGENTS.md)
- `dac_core`：[`dac_core/README.md`](./dac_core/README.md)
- `dac_web`：[`dac_web/README.md`](./dac_web/README.md)、[`dac_web/doc/AGENTS.md`](./dac_web/doc/AGENTS.md)、[`dac_web/doc/GOALS.md`](./dac_web/doc/GOALS.md)
- `rdb`：[`rdb/ReadMe.md`](./rdb/ReadMe.md)、[`rdb/docs/GOALS.md`](./rdb/docs/GOALS.md)、[`rdb/docs/GOALS-METAHUB.md`](./rdb/docs/GOALS-METAHUB.md)
- TC：[`rdb/tc-backend/README.md`](./rdb/tc-backend/README.md)
- MetaHub：[`rdb/mh-backend/README.md`](./rdb/mh-backend/README.md)
- `tpl`：[`tpl/README.md`](./tpl/README.md)、[`tpl/AGENTS.md`](./tpl/AGENTS.md)、[`tpl/doc/GOALS.md`](./tpl/doc/GOALS.md)

`GOALS.md` 描述方向与设想，不代表已经实现；发生冲突时以代码和根 `AGENTS.md` 为准。

## 已知缺口

- `dac_web`、`tpl` 尚未接入 TC；资源存储与搜索仍是各自实现。
- 认证配置不统一：`dac_web` 使用 `KEYCLOAK_*`，TC 使用 `KC_*`；TC 任务中的 realm 也与 `dac_web`、`tpl` 不同。
- 审阅状态不统一：`dac_web` 使用 `Registered/Approved/Rejected/Deleted`，TC 使用 `draft/pending_review/published/archived`。
- 缺少统一的前端外壳、API client 和跨应用导航。
- 根 workspace 未覆盖 TPL/RSP 后端和 RSP 前端。
- `tpl` 尚无认证与权限模型。

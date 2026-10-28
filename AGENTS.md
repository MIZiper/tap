# AGENTS.md

面向后续开发者与 AI 代理。先读根 [`ReadMe.md`](./ReadMe.md)，再按任务读取对应子模块文档。

## 1. 工作前必读

1. 根 `ReadMe.md`：仓库地图、固定端口、启动方式。
2. 子模块文档：
   - `dac_core/README.md`
   - `dac_web/README.md`、`dac_web/doc/AGENTS.md`
   - `rdb/ReadMe.md`、`rdb/tc-backend/README.md`、`rdb/mh-backend/README.md`
   - `tpl/AGENTS.md`、`tpl/README.md`
3. `GOALS.md` 只表示方向与设想，不代表已实现。
4. 文档与代码冲突时以代码为准；根文档只负责跨应用约定，子模块细节以子模块文档为准。

注意：`dac_web/doc/AGENTS.md` 中有旧的硬编码绝对路径（如 `/home/pi/dac_web/`），不要据此判断当前 workspace 路径。

## 2. 仓库拓扑与子模块规则

```
tap/
├── dac_core/     # miz-dac，共享 Python 库
├── dac_web/      # miz-dac_web + @mizip/dac_web
├── rdb/          # miz-metahub + @mizip/metahub，miz-tc + @mizip/tc
└── tpl/          # tpl-backend/tpl-frontend，rsp-backend/rsp-frontend（暂缓）
```

- 四个子模块都是独立 Git 仓库，根仓库只记录 submodule 指针。
- 修改子模块后，必须先在子模块内提交并推送，再回根仓库更新指针。不要在根仓库只改子模块文件而忘记提交子模块。
- 不要从根仓库直接“修复”子模块的 Git 历史；跨仓库改动按子模块分别提交。
- 根 `uv.lock`、`pnpm-lock.yaml` 属于协同开发配置。依赖或 workspace 变化后要同步更新并一起提交。
- 当前根工作区可能显示这两个锁文件为未跟踪；提交前先确认是否应纳入版本控制，不要误删。

## 3. Workspace 边界

### uv

根 workspace 成员：

```text
dac_core
dac_web/backend
rdb/mh-backend
rdb/tc-backend
```

不在根 workspace：

```text
tpl/tpl-backend
tpl/rsp-backend
```

因此：

- 根目录 `uv run ...` 只对上述四个成员可靠。
- TPL/RSP 后端必须在各自目录执行，使用各自 `uv.lock`。
- 新增跨包依赖时，先确认目标项目是否在根 workspace；调整 `[tool.uv.workspace].members` 与 `[tool.uv.sources]` 后重新 `uv lock`。
- 根 `[tool.uv.sources]` 当前映射 `miz-dac`、`miz-dac-web`、`miz-metahub`、`miz-tc`，不要随意改成 PyPI 版本。

### pnpm

根 workspace 成员：

```text
dac_web/dac-frontend   @mizip/dac_web
rdb/mh-frontend        @mizip/metahub
rdb/tc-frontend        @mizip/tc
tpl/tpl-frontend       tpl-frontend
```

不在根 workspace：

```text
tpl/rsp-frontend
```

根 `overrides` 已为 `@mizip/dac_web`、`@mizip/metahub`、`@mizip/tc` 配置 `workspace:*`。下游新增依赖时应使用 `workspace:*`，不要复制源码：

```json
{
  "dependencies": {
    "@mizip/tc": "workspace:*",
    "@mizip/metahub": "workspace:*"
  }
}
```

如果要把 RSP 前端接入共享包，先把 `tpl/rsp-frontend` 加入根 `pnpm-workspace.yaml`，再添加依赖并重新 `pnpm install`。

## 4. 固定端口与本地配置

固定端口来自子模块 `.vscode/tasks.json`：

| 应用 | PostgreSQL | 后端 | 前端 |
|---|---:|---:|---:|
| `dac_web` | 5431 | 8031 | 5131 |
| `rdb` TC | 5430 | 8030 | 5130 |
| `rdb` MetaHub | 5433 | 8033 | 5133 |
| `tpl` | 5437 | 8037 | 5137 |
| `tpl` RSP（暂缓） | 5439 | 8039 | 5139 |
| Keycloak | - | 8080 | - |

规则：

- 优先使用根 `.vscode/tasks.json` 或子模块的 VS Code 组合任务启动整套服务；根任务包含 `DAC`、`TC`、`MetaHub`、`RDB: Full`、`TPL`，不包含 RSP 和 Keycloak 容器。
- 根任务复制了子模块的启动命令（VS Code 不会跨子模块共享 tasks），修改端口时必须同步更新根任务、对应子模块任务、Vite 配置和根 `ReadMe.md`。
- 前端通过 `VITE_API_PORT` 连接固定后端端口。
- TC 通过 `TC_METAHUB` 找 MetaHub，默认 `localhost:8033`。
- 本地测试前如存在代理，先执行：
  ```bash
  unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
  ```
- PostgreSQL 版本：`dac_web` 和 `tc` 的 schema 使用 `uuidv7()`，需要 PostgreSQL 18+；TPL/RSP 使用 `gen_random_uuid()`，较旧版本也可运行。建议统一使用 PostgreSQL 18。

## 5. 跨应用复用规则

目标不是复制相似功能，而是建立稳定边界。

### 5.1 Python 库与服务

- `dac_core` 是唯一共享 Python 库。`dac_web` 通过 `miz-dac` 使用它。
- TC 与 MetaHub 是可独立运行的服务。下游优先通过 HTTP API 复用，不要直接 import 其他应用的内部模块。
- 如确需新增共享 Python 包，先明确包名、版本、workspace 关系和发布方式，不要从 `tpl`/`dac_web` 反向依赖 `rdb` 私有代码。
- 跨应用数据交换使用 JSON/HTTP 契约；后端 Pydantic 模型是 API 边界，不要把数据库行对象直接暴露出去。

### 5.2 通过 TC 复用资源存储

`dac_web`、`tpl` 的目标是成为 TC 实例。优先复用：

- 节点 CRUD：`tc/routes/nodes/*`
- 类型描述：`tc/content_types/registry.py`
- 跨类型检索/聚合：`POST /api/nodes/query`、`POST /api/nodes/aggregate`
- 文件：`/api/files`
- 标签：MetaHub + `tc/services/metahub.py`
- 权限与审阅：`tc/auth/*`

扩展 TC 类型时：

1. 后端在 `tc/content_types/registry.py` 新增 `TypeDescriptor`。
2. 用 `FieldSpec` 声明可检索/可聚合路径与稳定 `key`，这些 key 是跨类型比较的契约。
3. 需要内容升级时提供 `upcast`。
4. 前端在 `tc-frontend/src/modules/index.ts` 注册 `BaseProcessor`（viewer/editor/typeViewer）。
5. 通过 `ui.component` 关联前后端渲染器。
6. 不要 fork TC 的节点表或查询实现；下游差异放在 `nodes.content` JSONB 或应用自己的数据库中。

TC 的 descriptor 只负责投影，不限制 `content` 的原始结构。写入时会投影到 `node_fields`，并额外索引文本叶子到 `_content`。

### 5.3 前端复用

- 复用 TC/MetaHub 时，通过包导出使用，不要复制 `src/`：
  - `@mizip/tc` 导出 API、类型、`registry`、`BaseProcessor`、`NewResource`、`ResourceItem`、`Pagination` 等。
  - `@mizip/metahub` 导出标签选择相关组件与 API。
- 前端包同时提供 SPA 构建与库构建：
  - `pnpm build`：应用构建。
  - `pnpm build:svelte`：Svelte 包构建，输出 `dist-lib`。
- 下游扩展内容类型时注册自己的 processor，不要把类型判断硬编码进 TC 页面。
- 保持 Svelte 5 与路由约定；`dac_web` 大量使用 runes（`$state`、`$derived`、`$effect`），没有 Svelte stores。

## 6. 数据与存储边界

| 应用 | 默认数据库 | 主要存储 | 说明 |
|---|---|---|---|
| `dac_web` | `dac` | `nodes`、`histories`、`publishes`（已废弃） | 项目配置 JSONB；文件存储与 PostgreSQL 双模式，由 `DBSTORE` 控制 |
| TC | `tc` | `nodes`、`node_tags`、`files`、`data_types`、`field_keys`、`node_fields` | schema 启动时自动应用，幂等 |
| MetaHub | `metahub` | `clients`、`categories`、`tags`、`registered_nodes` | 标签树、下游客户端与节点登记 |
| TPL | `tpl` | `documents` | `plan_document` / `execution_document` 为 JSONB |
| RSP | `rsp` | 16 张 FMEA 表 | 暂缓；`rpn` 是生成列 |

约束：

- 每个应用使用独立数据库，不要跨库直接查询。
- TC 的 `nodes.content` 不强制 schema；需要检索的字段必须通过 descriptor 投影。
- `dac_web` 与 TC 都有 `nodes` 表，但结构、权限、状态含义不同，不能直接共用同一张表或数据库。
- `tpl` 的文档只存实例数据，行为在 `tpl-frontend/src/lib/` 的注册类中（fields/values/structs/transforms）。不要把这些行为搬到后端或写死到 JSON 中。
- `rsp` 的 `rpn` 为生成列，禁止直接 insert/update。
- asyncpg 连接必须注册 UUID/JSONB codec；新增连接池时沿用各项目现有 `init` 方式。

## 7. 认证、权限与审阅

### Keycloak

- 所有后端在启用认证时必须自行验证 JWT；前端 `keycloak-js` 与角色判断只用于 UI。
- 统一配置：realm `master`、client `miz`。
  - `dac_web`：`KEYCLOAK_REALM=master`、`KEYCLOAK_CLIENT_ID=miz`
  - `tpl`：已接入相同配置，使用 JWT 保护文档创建与 TC 关联接口
  - TC：`KEYCLOAK_REALM=master`、`KEYCLOAK_CLIENT_ID=miz`
- 环境变量统一使用：
  - `KEYCLOAK_URL`
  - `KEYCLOAK_URL_PUBLIC`
  - `KEYCLOAK_REALM`
  - `KEYCLOAK_CLIENT_ID`
  - `KEYCLOAK_CLIENT_SECRET`（仅 `dac_web` 当前使用）
- TC 的内部 Settings 字段为 `keycloak_*`；对外 `/api/config` 仍返回 `kc_url`、`kc_realm`、`kc_client_id`，不要为改字段名而破坏前端契约。
- 新增应用时必须复用上述命名与 realm/client，不要重新引入 `KC_*` 或独立 realm。

### 角色

TC 的参考角色：

```text
nodes:create
nodes:read_all
nodes:edit_any
nodes:review
nodes:visibility:<name>
```

- TC 从 realm roles 与 `resource_access.miz.roles` 合并角色（当前 client 为 `miz`）。
- 节点创建者始终可编辑/查看自己的节点，即使没有全局角色。
- `dac_web` 使用 `tap:admin` 作为管理角色（`admin_role` 默认值），合并读取 realm roles 与 `resource_access.miz.roles`。
- 应用管理角色统一为 `tap:admin`；内容/节点操作使用 TC 的 `nodes:*` 角色。

### 审阅状态

- TC：`draft`、`pending_review`、`published`、`archived`，由 `nodes:review` 控制变更。
- `dac_web` 的 `publishes.status` 已废弃；发布状态统一以外部关联系统的状态为准。
- 应用只缓存外部状态，并根据 frozen 集合判断只读；不要重新引入第二套发布状态。
- `tpl` 已接入 Keycloak JWT；未配置时保持本地匿名使用，尚无细粒度角色与审阅模型（admin JSON 编辑器按本地可信工具处理）。

### 敏感信息

- `dac_web` 的 `creator_signature` 类似密码，禁止返回前端、展示或写入日志。
- Keycloak client secret、admin token、数据库密码不得提交或打印。
- 不要在文档、测试快照或错误响应中暴露 token。

## 7.1 外部链接与 frozen

- 用户在 TC 中创建统一的 `SourceItem.v00`，再从 `dac_web` / `tpl` 关联；source 类型（`dac_web.project`、`tpl.plan`、`tpl.log`）不提供给用户直接创建。
- source 应用在 `PUT /api/nodes/{id}/source` 中带 `data_type`；TC 原子写入 source 字段并回填 `content_type`。解链时恢复 `SourceItem.v00`。
- source 应用使用 `GET /api/nodes/source-candidates` 搜索未链接的 Source Item。
- 关联后 TC 拥有发布元数据（标题、描述、标签、状态），应用不要回写这些字段；TC 详情页显示来源应用与跳转链接。
- 已链接节点的 content 由 source 控制；只有 `nodes:edit_any` 可以在 TC 中直接修改。
- TPL 的 `plan` 和 `log` 是两个独立链接、独立状态、独立 frozen。
- frozen 判定：`nodes.frozen = true` 或 status ∈ `TC_FROZEN_STATUSES`；应用侧 `EXTERNAL_FROZEN_STATUSES` 作为 fallback。frozen 节点拒绝 `/push/data`（423）。
- 已链接但外部状态不可确认时 fail-closed；未链接资源不受影响。
- `dac_web` frozen 时仍允许 Save As；Save As 不继承外部链接。
- 外部状态接口统一为 `GET/PATCH /api/nodes/{id}/state` 与 `GET/PUT/DELETE /api/nodes/{id}/source`。
- 一个应用实例只配置一种 `EXTERNAL_SYSTEM`；当前实现 `tc`，WindChill 后续通过同一 provider 契约接入。
- `dac_web` 的旧 `publishes`/`PubStatus` 已废弃，不再作为状态源。
- `dac_web` 的 `Push data to TC` action 将 `TimeData`/`FreqDomainData` 转为 `Tabular.v00`，通过 `PUT /api/nodes/{id}/push/data` 推送；`node_contributions` 记录来源，允许多个项目向同一 item 追加。push 权限沿用 `check_edit_node`（creator 或 `nodes:edit_any`），不新增角色。

## 8. 各应用关键机制

### dac_core

- 核心模型：`DataNode`、`ActionNode`、context、scenario。
- 项目保存为 `*.dac.json`；scenario 用 YAML 定义可用类型/动作/快捷动作。
- 公共基础类型属性自动序列化，私有属性以 `_` 开头不保存。
- 修改核心序列化或节点解析时要考虑桌面端、Web 端和脚本用法。

### dac_web

- 每个分析会话是一个独立子进程，由 UUID 标识；主后端通过 `/app/*` 反向代理到子进程。
- 会话生命周期：`/api/new`、`/api/load`、`/api/term`、`/api/save`；前端 `onDestroy` 负责清理。
- `DBSTORE=true` 使用 PostgreSQL，否则使用 `storage/projects`、`storage/projects_save` 文件。
- 导入桌面项目有两阶段替换/审阅流程：`dac_web/import_replace/*` 与 `rules/*`。
- SSE 用于动作执行进度；`/mpl` 与 WebAgg 用于图形。
- 前端路由使用 `sv-router`；不使用 Svelte stores。

### TC

- 启动时自动执行幂等 schema 并 seed `data_types`。
- 内容类型在 `content_types/registry.py` 集中定义；不要为单个下游在路由中硬编码类型逻辑。
- 查询/聚合基于 `node_fields`，字段 key 是跨类型契约。
- 文件上传后通过 `file_id` 关联节点，文件可见性继承节点。
- 权限判断集中在 `auth/permissions.py` 和路由依赖中；新增路由必须复用这些依赖。

### MetaHub

- 标签按 client → category → 树状 tag 组织；`exposed` 决定下游是否可见。
- `registered_nodes` 记录下游节点，不存节点内容。
- TC 通过 MetaHub 展开父标签/后代标签，再查询自己的节点。

### TPL

- 后端只存 JSONB 文档；plan/execution 的校验和行为主要在前端类注册表。
- `required_executions = 0` 表示不计次数/无目标；计划甘特图会省略该步骤。
- 前端支持离线 JSON 导入导出；没有后端同步队列。
- 暂停/紧急转移会拆分 run，具体规则见 `tpl/AGENTS.md`；改动前先读该文件。
- RSP 暂缓，除非明确要求，不要扩展 RSP。

## 9. 常用命令

```bash
# 根依赖
uv sync
pnpm install

# dac_core
uv run pytest dac_core/tests

# 后端导入检查
uv run python -c "import dac_web.main"
uv run python -c "import tc.main"
uv run python -c "import metahub.main"

# TPL / RSP 独立环境
cd tpl/tpl-backend && uv run python -c "from tpl.main import app"
cd tpl/rsp-backend && uv run python -c "from rsp_backend.main import app"

# 前端检查（根 workspace）
pnpm --filter @mizip/dac_web check
pnpm --filter @mizip/tc check
pnpm --filter @mizip/metahub check
pnpm --filter tpl-frontend check

# RSP 前端
cd tpl/rsp-frontend && pnpm check
```

没有统一的根 lint/test 脚本；按应用分别验证。

## 10. 修改检查清单

跨应用功能：

1. 明确复用边界：共享库 / 服务 API / 前端包 / 仅复制契约。
2. 确认目标项目属于哪个 workspace；更新成员、依赖、锁文件。
3. 优先在 TC 增加 descriptor/processor，而不是复制节点表或查询逻辑。
4. 如涉及认证/审阅，先确认角色与状态映射，不要静默引入第二套语义。
5. 跑单应用检查：
   - 后端导入检查
   - 前端 `check` / `build`
   - 相关集成测试
6. 子模块内提交并推送，再更新根仓库 submodule 指针。
7. 若行为或约定变化，同步更新子模块 AGENTS/README 和根文档。

## 11. 当前已知缺口

- `dac_web`、`tpl` 已支持 TC 链接，但尚未成为完整 TC 实例。
- Keycloak env 命名、realm/client 与角色命名已统一为 `nodes:*` 和 `tap:admin`。
- 应用不再维护自己的发布状态；状态以 TC 为准。
- 缺少统一前端 shell、API client、跨应用导航。
- TPL/RSP 后端与 RSP 前端不在根 workspace。
- `tpl` 已支持 Keycloak 登录与 `nodes:create` / `nodes:edit_any` / `tap:admin` 基础权限，尚无审阅模型。
- `dac_web` 与 TC 都使用 `uuidv7()`，依赖 PostgreSQL 18+。

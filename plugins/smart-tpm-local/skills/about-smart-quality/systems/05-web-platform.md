# SmartQuality APP V2 Web 端（检测平台中枢）

> 交接文档模块 · 系统篇 05
> 覆盖对象：仓库 `smart_tpm_web_v2`，内部标题 **"Smart TPM Edge SmartQuality System"**——整套产线声学质量检测的 **Web 平台**：React 前端 + FastAPI 后端 + MySQL/Redis/InfluxDB/MinIO 存储 + Nginx。
> 一句话定位：**这是整个 SmartQuality 生态的"中枢后台"**。产线现场采集（安卓/桌面客户端，见[App 客户端](./04-app-client.md)）、数据同步（见[数据同步服务](./02-data-sync-service.md)）、AI 分析与插件（见[移动套装 / AI 分析链路](./06-mobile-suite.md)）都围绕它转；它管人、管产线、管数据集、管算法、管检测记录，并对外暴露给 AI 的接口层也长在它身上。

术语首次出现即就地解释；核心业务概念可到[核心概念](../reference/04-key-concepts.md)下钻，名词另见[术语表](../glossary/terms.md)。整体分层见[整体架构](../reference/02-architecture.md)。

---

## 第一段 · 系统定位与总览

### 1.1 它在生态里的位置

用一句话给接手人定位：**前端是"人用的后台"，后端是"所有数据的出入口"。** 现场客户端只负责录音和录入，真正的产品/工位/算法/数据集配置、检测记录归档、算法评测、以及给 AI 插件供数，全在这套 Web 平台。它是"Total Productive Maintenance（TPM，全员生产维护）"命名下的声学质检平台，产品对外品牌名 **SmartQuality**。

TPM = Total Productive Maintenance，是本系统的历史命名前缀（`smart_tpm_*`），实际业务已聚焦"声学质量检测"，见 [产品概览](../reference/01-product-overview.md)。

### 1.2 仓库物理结构（顶层目录）

| 目录 | 角色 | 说明 |
|------|------|------|
| `smart_tpm_edge_web_v2/` | **React 前端** | UmiMax 工程，产物是静态站，Nginx 托管 |
| `smart_tpm_edge_api_v2/` | **FastAPI 后端** | 主服务，含 REST + MCP + Celery 预计算 |
| `smart_tpm_sync_api/` | 同步上报 API | 独立小服务（现场→中心的同步告警/上报），见[数据同步](./02-data-sync-service.md) |
| `tpm-realtime-app/` | 实时链路组件 | `ui/` + `service/` |
| `poc-precompute-link/` | 预计算 POC | 试验性目录，非主线 |
| `docker-compose.yml` / `docker-compose.deploy.*.yml` | 编排 | 开发用 profile；`.121/.175/.234` 为三套现场部署 |
| `deploy_121.py` / `deploy_175.py` / `deploy_234.py` | 发布脚本 | 按环境推镜像/起栈 |
| `docs/` | 设计与测试文档 | MCP 各模块 plan、测试计划、拓扑 SVG |
| `sql/`、`smart_tpm_edge_api_v2/alembic/` | 数据库脚本与迁移 | |

> `CLAUDE.md` 规定项目内正文文档统一写入外部 Obsidian 库，仓库里的 `.md` 只保留 `README`/`CLAUDE.md` 与 skill 配置——接手人找不到设计文档时先去 Obsidian。

### 1.3 技术栈总表

| 层 | 选型 | 一句话说明 |
|----|------|-----------|
| 前端框架 | **UmiMax**（`@umijs/max` 4）+ React 18 + TypeScript | UmiMax：蚂蚁的企业级 React 应用框架，内建路由/请求/权限/国际化 |
| UI | TailwindCSS + **Radix UI** + `lucide-react` 图标 | Radix：无样式可访问组件原语（Dialog/Select/Tabs 等） |
| 数据请求/表格 | **TanStack Query** v4 + TanStack Table v8 | React Query：服务端状态缓存与请求管理 |
| 全局状态 | **zustand**（仅 `stores/workstation.ts`） | 轻量状态库；本项目状态大多托管在 React Query，zustand 只存"当前工位" |
| 图表/可视化 | ECharts、Plotly、Recharts、**wavesurfer.js** | wavesurfer：音频波形渲染，用于声学记录回放 |
| 流程编辑器 | **`@xyflow/react`**（React Flow）+ elkjs 布局 | 拖拽式有向图，用于"检测流程/算法流程"可视化编排 |
| 重计算 | Web Worker（`spectrogramWorker` 频谱、`dataProcessingWorker`） | Web Worker：浏览器后台线程，避免频谱计算卡 UI |
| 导出 | `xlsx`、`jspdf`、`html-to-image` | 记录/报表导出 Excel/PDF/图片 |
| 后端框架 | **FastAPI** 0.104 + uvicorn/gunicorn | Python 异步 Web 框架 |
| ORM/迁移 | **SQLAlchemy 2.0**（async + aiomysql）+ **Alembic** | Alembic：数据库版本迁移工具 |
| 异步任务 | **Celery** + Redis broker（音频预计算 worker + beat 定时） | Celery：分布式任务队列 |
| 存储 | MySQL 8 · Redis 7 · InfluxDB 2.7 · MinIO(S3) | 分工见第二段 2.4 |
| 网关 | Nginx（前端静态托管 + 反代） | |
| 旁路基建 | RabbitMQ、Prometheus、Grafana | compose 里声明，监控/消息用 |

### 1.4 端口速查（通用默认，非现场实际）

| 服务 | 端口 | 备注 |
|------|------|------|
| Web 前端（容器/dev） | 8005 / dev 8006 | 生产由 Nginx 80 托管 |
| FastAPI 后端 | 8000 | `API_PREFIX=/api/v2`；`/docs` 为交互文档 |
| MySQL | 3306 | |
| Redis | 6379 | db0 缓存 · db1 Celery broker · db2 结果 |
| InfluxDB | 8086 | 时序库 |
| MinIO | S3 兼容（现场自定端口） | 对象存储 |
| Grafana / Prometheus | 3000 / 9090 | 监控 |

> 具体主机地址、密钥不在本文；现场配置见各环境 `.env.production.*`（不入公开库）。

---

## 第二段 · 逐层详解

### 2.1 前端工程结构与路由

工程根 `smart_tpm_edge_web_v2/src/`，主要子目录：

```
src/
  app.tsx              # 运行时入口（getInitialState/权限初始化）
  access.ts            # 权限点定义（canViewXxx / canCreateXxx …）
  layouts/MainLayout   # 主框架布局（侧边菜单、顶栏、版本号 Version x.y.z）
  pages/               # 按业务域分的页面（见 2.2）
  services/            # 后端接口封装（每个业务域一个 *.ts，约 50 个）
  stores/workstation.ts# zustand：当前选中工位
  hooks/ components/ ui/ wrappers/  # 复用 hook、业务组件、Radix 封装、HOC
  workers/             # 频谱 / 数据处理 Web Worker
  locales/             # 国际化（默认 zh-CN，支持切换）
  config/ data/ types/ utils/ lib/
```

**路由**：不用 Umi 的文件约定，而是集中在 **`.umirc.ts` 的 `routes` 数组**里手写声明。特征：

- 登录 `/login`、`/no-access`、`/404`、OAuth 授权页 `layout: false`（不套主框架）。
- 一批"全屏工具页"也 `layout: false` 独立打开：数据集标注 `/detection/dataset/:id/annotate/:detectId`、检测记录标注（只读）、测试记录检测视图、以及两个**流程编辑器**（`.../flow-editor/...`，检测流程 / 算法参数流程）。
- 其余业务页挂在 `@/layouts/MainLayout` 下，每条路由带 `access: 'canViewXxx'` 做**权限门禁**——权限点定义在 `access.ts`，由登录态与后端权限接口决定。

**状态管理**：以 **React Query 缓存服务端数据**为主线（列表/详情/表单提交都走 `services/*` + query hooks），全局仅用 zustand 保存"当前工位"这一跨页共享态。国际化、初始态、请求由 UmiMax 插件（`locale`/`initialState`/`request`）统一接管。

### 2.2 页面与功能清单（核心业务模块）

按 `.umirc.ts` 路由与 `pages/` 目录整理，`PlaceholderPage` 标记的是**占位未实现**：

| 业务域 | 页面/功能 | 路由前缀 | 状态 |
|--------|-----------|----------|------|
| 仪表盘 | Dashboard 概览 | `/dashboard` | 有 |
| **检测（Detection）** | 数据集列表/建/改/导入/**标注**；检测工位；检测字典；**检测记录**（列表+只读标注）；声学记录 | `/detection/*` | 数据集/工位/字典/记录 有；`/detection/acoustic` 占位 |
| **算法（Algorithm）** | 算法 API、**算法参数**（含流程编辑器）、**测试任务**、**测试记录**（详情/对比/检测视图）、模型产物（ModelArtifact）、评分（Rating） | `/algorithm/*` | 有 |
| **数据（Data）** | **检测流程 DetectFlow**（建/改/流程编辑器/执行日志）、特征库 FeatureStore（任务/导入/报告） | `/data/*` | 有 |
| 生产（Production） | 项目 Project、产品 Product、装配线 AssemblyLine；生产计划 | `/production/*` | 前三者有；`/production/plan` 占位 |
| 设备（Device） | 设备列表、PLC、智能 PLC、智能设备 | `/device/*` | 有（PLC：可编程逻辑控制器，采集/控制现场设备） |
| **系统（System）** | 用户、角色、权限、审计日志、**MCP 审计**、API Key、归档 Archive、工位、同步配置（含告警） | `/system/*` | 有 |
| 设置（Settings） | 向量库、AI API、语言配置、Tinia 配置、预计算配置 | `/settings/*` | 有 |
| 账户/集成 | 已连接应用（MCP OAuth 授权管理）、BI 报表 iframe 嵌入、OAuth 授权页 | `/account`、`/bi/embed`、`/oauth/authorize` | 有 |
| 质量/维护/报表 | 质量控制、维护计划、报表 | `/quality`、`/maintenance`、`/reports` | **占位（PlaceholderPage）** |

术语落点：**数据集 / 标注 / 检测记录 / 测试任务 / 检测流程 / 算法参数**这些核心概念的业务含义见[核心概念](../reference/04-key-concepts.md)第二段。

### 2.3 后端 FastAPI 架构与接口分层

后端根 `smart_tpm_edge_api_v2/app/`：`api/`（路由）、`services/`（业务，约 55 个）、`models/`（SQLAlchemy 模型）、`schemas/`（Pydantic 校验）、`core/`（配置/数据库/鉴权/加密/MinIO）、`mcp/`（AI 插件服务）、`precompute/`（Celery 音频预计算）、`middleware/`、`alembic/` 迁移。

主入口 `app/main.py` 把三块路由挂到应用：

| 挂载 | 前缀 | 面向谁 / 鉴权 |
|------|------|----------------|
| `api_router`（`api/v2/`，约 40+ 子路由） | `/api/v2/*` | **Web 前端**主接口，JWT 登录态 + 权限 |
| `well_known_router` | 根路径 `/.well-known/*` | OAuth 元数据发现（协议规定不带前缀） |
| `file_v1_compat_router` | `/api/v1/file/*` | **老麦克风盒子上传 / App 回放下载**兼容（路径写死无法迁移） |

`api/v2/` 内部又分四类子域：

| 子域 | 路径 | 鉴权 | 用途 |
|------|------|------|------|
| 主业务 | `/api/v2/<模块>` | JWT + 权限点 | 前端用；模块含 auth/dataset/project/product/user/role/permission/algorithm-*/test-*/detect-*/device/plc/feature-store/… |
| **appapi** | `/api/v2/appapi/v1/*` | **X-API-Key**（scope `appapi:*`）+ 部分带 `X-Workstation-Id` | 与老系统 `/api/v1/*` **一一对齐**的兼容层（登录/产品/工位/质量检测/传感器实时/品牌配置） |
| **ext** | `/api/v2/ext/*` | OAuth `ext:*` scope | 给 **Tinia 分析插件** `smart_quality_v2` 接数据集/工位/检测记录/音频 |
| **open** | `/api/v2/open/*` | X-API-Key（+ 工位相关带 `X-Workstation-Id`） | 对外开放接口：algorithm-flow/dataset/test-record/detect-record/data |

> **API Key / Scope / X-Workstation-Id**：API Key 是给非浏览器接入方（老系统、插件、外部）的令牌，scope 限定它能碰哪些模块；工位相关接口必须额外声明"操作哪个工位"，这直接关联下面的**工位分库**。

**MCP 服务层（`app/mcp/`）**：后端内嵌一套 **MCP（Model Context Protocol，让 AI 客户端调用工具的协议）** 服务，模块覆盖 `datasets / detect_flows / test_tasks / algorithms / files / marks / metadata / trace / config_lookup / extras`。它把平台数据以"工具"形式暴露给 Claude 等 AI 客户端，是 [AI 插件层](./06-mobile-suite.md)的服务端。审计走 `/system/mcp-audit` 页面；OAuth 授权由 `oauth` 路由 + 前端"已连接应用"页管理。

前端 `services/*.ts` 与后端路由基本一一对应（如 `dataset.ts↔/dataset`、`testTask.ts↔/test-task`、`detectRecord.ts↔detect-record`），接手时"改一个功能"沿这条命名链即可从页面追到接口再追到 service 与 model。

后端 `services/` 层还有几个"底座类"文件值得先认识：`base.py`（service 基类）、`cache.py`（Redis 缓存封装）、`file_storage.py`（MinIO presigned URL）、`remote_db.py`（跨库/远程库访问）、`message_queue.py`（RabbitMQ）、`audit.py`（操作审计）、`feature_extraction.py` / `spectro_params.py` / `audio_visualization.py`（声学特征与频谱）、`vector_db.py` / `tsne_service.py`（向量库与降维可视化，配合 Settings 的向量库页与特征库）。改任何业务前先看它依赖了哪几个底座。

### 2.3.1 一条检测记录的端到端流转（据代码域模型）

给接手人一条"主动脉"心智模型，帮助把上面分散的模块串起来：

1. 现场客户端（安卓/桌面）在某工位录音，音频经 `/api/v1/file/*` 兼容接口或 appapi 上传，落 MinIO `smart-tpm-audio` 桶。
2. 客户端把这次质检的结果/元数据经 appapi（`detect_app` / `detect_station_app`）写入——**记录落到该工位的分库**，不是主库。
3. Celery `audio-worker` 拉取录音做预计算（Tinia 特征/频谱），结果写 MinIO `audio-precompute` 桶。
4. Web 前端在"检测记录"页按工位（`X-Workstation-Id`）读分库列表；打开标注页时用 wavesurfer 回放录音、用 Web Worker 算频谱、按"检测字典"给样本打标签。
5. 标注后的样本可归入"数据集"，进而喂给"算法参数 / 测试任务"做评测，产出"测试记录"（含对比视图），模型产物入 MinIO `model-artifacts`。
6. 全过程的读写既可被前端消费，也可经 MCP / open / ext 子域被 AI 客户端与 Tinia 插件消费。

这条链路对应的核心概念（工位、数据集、标注、测试任务、模型产物）逐一解释见[核心概念](../reference/04-key-concepts.md)。

### 2.4 数据存储分工

| 存储 | 承担 | 关键点 |
|------|------|--------|
| **MySQL（主库）** | 业务主数据：用户/角色/权限、产品/项目/装配线、工位、数据集、算法参数/流程、测试任务/记录、字典、审计、系统配置 | 主库名由 `DATABASE_URL` 解析（`MASTER_SCHEMA`），所有主库模型带 `schema=MASTER_SCHEMA` |
| **MySQL（工位分库）** | **每个检测工位一套独立数据库**，存该工位的检测/质检记录 | 见 2.5 |
| **Redis** | 缓存、会话、Celery broker（db0 缓存 / db1 broker / db2 结果） | `maxmemory-policy allkeys-lru` |
| **InfluxDB** | 时序数据库，规划承接传感器/声学时序（bucket `sensor-data`） | 基建已就位、配置项存在；**当前 v2 后端代码对其引用有限**，时序落点实际多走 MinIO 预计算 + MySQL——接手时勿假设它是热路径 |
| **MinIO（S3 兼容对象存储）** | 录音音频、模型产物、预计算结果 | 三个桶：`smart-tpm-audio`（录音）、`model-artifacts`（模型）、`audio-precompute`（预计算结果）；用 boto3 生成 **presigned URL**（带签名的临时下载直链）供客户端拉取 |

InfluxDB：时序数据库，擅长高频写入的传感器读数。MinIO：自建的、兼容亚马逊 S3 API 的对象存储，本项目当"文件仓库"用。

**预计算链路**（`precompute/` + compose 的 `audio-worker`/`scheduler-beat`）：录音入 MinIO 后，Celery worker 走 **Tinia SDK**（声学分析 SDK，见[移动套装](./06-mobile-suite.md)）算频谱/特征，结果回 MinIO `audio-precompute` 桶；`scheduler-beat` 定时清理过期结果（`PRECOMPUTE_RESULT_TTL_DAYS`）。Tinia 拒绝本地 license 时有 librosa 兜底开关（`PRECOMPUTE_FALLBACK_LIBROSA`，默认关）。

### 2.5 工位分库 / 检测记录分库（重点机制）

这是本平台**最容易踩坑**的架构点，接手必读：

- `Workstation`（工位）表的每一行都带一组**自己的数据库连接**：`db_host / db_port / db_name / db_user / db_password`（密码用 `WORKSTATION_DB_ENCRYPT_KEY` 加密存储）。
- 也就是说——**主库存"工位有哪些、连哪个库"，每个工位的检测记录存在它自己的那套 MySQL 库里**。这就是"工位分库 / 检测记录分库"。
- 后端 `core/database.py` 的 `DatabaseManager` 维护一个**按工位缓存的引擎池**：`_workstation_engines / _sessionmakers`，带空闲超时（`WORKSTATION_ENGINE_IDLE_TIMEOUT` 默认 1800s）、连接池尺寸（`WORKSTATION_POOL_SIZE=20` / overflow 50）。
- 缓存以 `db_url` 为指纹：**工位改了库地址时会重建引擎**。历史上出过"多 worker 下缓存不一致，数据一会有一会没有"的 bug（git `caa7d11` 修复：db_url 变更即重建）——多进程部署时留意这类缓存一致性问题。
- 前端 zustand `stores/workstation.ts` 存"当前工位"，请求经 `X-Workstation-Id` 头把操作路由到对应分库；appapi / open 子域也靠这个头选库。

---

## 第三段 · 构建发布、已知缺陷与接手指引

### 3.1 构建与发布流程

**前端**：`npx max build`（脚本 `build` / `build:test` / `build:staging` / `build:production`）产出 `dist/` 静态资源（`hash:true` 带内容哈希防缓存），多阶段 Dockerfile（`node:18-alpine` 构建 → `nginx:alpine` 托管）。dev 用 `npm run dev`（UmiMax dev server）。

**后端**：多阶段 `python:3.11-slim`，`Dockerfile.prod` 先装依赖（含 `vendor/` 里的 Tinia SDK 本地包），运行时非 root，gunicorn + uvicorn worker 起 FastAPI；Alembic 迁移随镜像打包（`alembic upgrade head`）。

**编排/环境**：`docker-compose.yml` 用 profile 区分 `development`（api+web+worker+beat）与 `production`（含 Nginx）。三套现场各有 `docker-compose.deploy.{121,175,234}.yml` + `deploy_{121,175,234}.py` 发布脚本 + `.env.production.{121,175,234}`。栈内含 MySQL/Redis/InfluxDB/RabbitMQ/Prometheus/Grafana/Nginx + api/web/audio-worker/scheduler-beat。

**版本号约定**（见仓库 `CLAUDE.md`）：每次提交把前端 patch 版本 +1，版本号写在 `src/layouts/MainLayout.tsx` 的 `Version x.y.z`（撰写时 **1.6.93**）。

**质量工具**：前端 ESLint + Prettier + `tsc --noEmit` + Playwright e2e（`e2e/*.spec.ts`，覆盖 auth/dashboard/navigation/language 等）；后端 pytest + flake8 + mypy + black/isort，测试分 `unit/integration/performance/security/websocket/precompute` 多类。

**数据库迁移**：主库结构变更走 Alembic（`alembic/`，`alembic upgrade head`），工位分库的结构同步另有机制（`precompute/sync_db.py` 等），迁移新字段时注意主库与分库两侧都要覆盖。配置开关集中在 `app/core/config.py`（存储地址、预计算 TTL、工位库加密 key、CORS、Celery broker 等），改行为前先通读一遍它，避免漏改环境变量。

### 3.2 已知缺陷与后续可考虑方向（弱措辞）

- **占位页**：质量控制、维护计划、报表、生产计划、声学记录（`/detection/acoustic`）等仍是 `PlaceholderPage`，菜单可见但功能未落地，后续可视排期补齐。
- **InfluxDB 使用偏薄**：基建与配置齐全，但代码热路径引用有限；若要真正用时序库承接声学/传感器数据，需要先补齐写入与查询链路。
- **工位分库的多进程一致性**：引擎缓存曾出一致性 bug，横向扩 worker 时建议复核 `DatabaseManager` 的重建条件与空闲回收。
- **appapi/v1 兼容层是历史包袱**：为对齐老 `/api/v1/*` 保留了字段映射与路径写死（含 `/api/v1/file/*` 麦克风盒子接口），迁移老客户端前不宜删。
- **预计算 librosa 兜底**：`PRECOMPUTE_FALLBACK_LIBROSA` 默认关闭；启用兜底会与 Tinia 结果口径不同，评测数据一致性上需留意。
- **文档散在 Obsidian**：仓库内几乎不留正文文档，接手前先要到外部 Obsidian 库的访问权，否则只有代码可依。
- **两套 MCP marketplace**：业务 tool 与 MinIO 文件下载分属两个 plugin marketplace，且有 `-local`/`-test` 变体切换，接入 AI 客户端时容易配错环境（见 `CLAUDE.md`）。

### 3.3 接手人从哪读起代码

按"从看得见到看不见"的顺序：

1. **先跑起来**：读根 `README.md` + `docker-compose.yml`，用 `--profile development` 起栈，前端 8005/后端 8000/`/docs` 看接口全貌。
2. **前端主线**：`smart_tpm_edge_web_v2/.umirc.ts` 的 `routes`（一览有哪些页）→ `src/layouts/MainLayout`（菜单与框架）→ 挑一个业务域（如检测记录）从 `pages/Detection/Record` 追到 `services/detectRecord.ts`。
3. **权限模型**：`src/access.ts` + `src/app.tsx` 的 initialState，理解 `canViewXxx` 门禁怎么来。
4. **后端主线**：`smart_tpm_edge_api_v2/app/main.py`（三处 `include_router`）→ `app/api/v2/__init__.py`（40+ 子路由清单）→ 对着前端选的域看 `api/v2/<模块>.py` → `services/*` → `models/*`。
5. **必读的坑**：`app/core/database.py` 的 `DatabaseManager`（工位分库/引擎缓存）、`app/core/config.py`（存储与开关全景）、`app/models/workstation.py`（工位带库连接字段）。
6. **对外接口三子域**：`api/v2/appapi`（老系统兼容）、`api/v2/ext`（Tinia 插件）、`api/v2/open`（开放 API），配合 `core/api_key_auth.py` / `mcp_auth.py` 理解鉴权。
7. **AI 与预计算**：`app/mcp/modules/*`（给 AI 的工具）、`app/precompute/*` + compose 的 `audio-worker`/`scheduler-beat`（音频特征预计算），延伸到[移动套装 / AI 分析链路](./06-mobile-suite.md)。
8. **发布**：`deploy_175.py` + `docker-compose.deploy.175.yml` + `.env.production.175` 看一套完整现场部署怎么落地。

> 记忆锚点：**"前端是 UmiMax 路由表 + services 命名链，后端是三处 include_router + 工位分库。"** 抓住这两句，其余按业务域顺藤摸瓜即可。

---

延伸阅读：[整体架构](../reference/02-architecture.md) · [核心概念](../reference/04-key-concepts.md) · [技术栈](../reference/09-tech-stack.md) · [数据同步服务](./02-data-sync-service.md) · [App 客户端](./04-app-client.md) · [移动套装 / AI 分析链路](./06-mobile-suite.md) · [术语表](../glossary/terms.md)

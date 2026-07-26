# 技术栈 — 全系统一览

> SmartQuality（Smart TPM）不是单体应用，而是一组各司其职的仓库：Web 平台、数据同步、移动采集套装、声学算法后端、联网录音、任务协同后台、日报 App、以及把这些暴露给 AI 的 MCP 插件。每个仓库按自身场景选型，语言从 Python / Go / TypeScript 到纯 Markdown 都有。这份文档先给一张**主表**看全貌，再逐条讲**关键选型的理由**。

本文分三段：
1. **技术栈主表** —— 一行一个系统，横向对齐语言/框架/存储/通信/前端。
2. **逐系统说明** —— 每个系统补充选型背景与关键依赖。
3. **横切关注点** —— 跨系统共用的存储、鉴权、部署约定。

版本号取自各仓库的 `package.json` / `requirements.txt` / `go.mod` / README，可能随迭代变化，**以各仓库当前清单为准**。本文不写任何内网地址、端口、凭据。

---

## 第一段 · 技术栈主表

| 系统（仓库） | 语言 | 框架 | 存储 | 通信 | 前端 |
|---|---|---|---|---|---|
| Web 平台后端 (`smart_tpm_edge_api_v2`) | Python 3.11 | FastAPI + SQLAlchemy 2.0 + Alembic | MySQL 8 · Redis · InfluxDB · MinIO(S3) | REST / HTTP + MCP over HTTP | — |
| Web 平台前端 (`smart_tpm_edge_web_v2`) | TypeScript | React 18 + UmiJS Max | 浏览器本地态 (zustand) | REST + WebSocket | React / Radix UI / Tailwind |
| 数据同步服务端 (`sync-data/sync-server`) | Go 1.21+ | 自研引擎 + Gin(admin) | MySQL / PostgreSQL 目标库 · Redis Stream · MinIO | gRPC 双向流 | React 18 + Ant Design 5 |
| 数据同步客户端 (`sync-data/sync-client`) | Go 1.24+ | 自研引擎 + go-mysql | SQLite(断点) · MinIO | gRPC 双向流 | Vue 3 + Vite |
| 移动套装 (`SmartQuality-client`) | TypeScript | Electron 33 + React 18 + Express | PostgreSQL(TypeORM) · 本地文件 | REST + mDNS 发现 | React / Konva / Tailwind |
| 声学算法后端 (`ai_api_server_v2`) | Python | 自研 DAG 引擎 | MySQL · MinIO · Qdrant | REST (`/detect/execute`) | — |
| 联网录音 (`net_audio` / SmartAudio) | Go 1.23 | PortAudio + systemd | 本地环形缓冲 / 文件 | REST + WebSocket + mDNS | Vue 3(embed) |
| 任务协同后台 (`wxdevops-task-admin`) | Python 3.9 | Flask 3.0 + SQLAlchemy | MySQL | REST + 企业微信/云效 API | React 18 + Ant Design 5 |
| 日报 App 源码 (`framework_v3_smart_quality`) | Python | report_sdk / bestfunc_sdk | 平台托管(KV/Daily) | 平台 endpoint / DAG | HTML 模板 |
| MCP 插件仓 (`smart_quality_plugins`) | Markdown + JSON | Claude Code plugin spec | — | MCP over HTTP + OAuth 2.1 | — |

> "MCP" = Model Context Protocol，把后端能力以标准工具形式暴露给 AI 客户端的协议。详见 [12-ecosystem.md](./12-ecosystem.md) 与子百科 [about-smart-tpm-mcp](../../about-smart-tpm-mcp/SKILL.md)。

### 关键版本明细

下面是从各仓库清单里摘的、值得记住的锁定版本（可能随迭代变化，以清单为准）：

| 领域 | 依赖 | 版本 | 用在哪 |
|---|---|---|---|
| 后端框架 | FastAPI | 0.104 | Web 平台后端 |
| 后端 ORM | SQLAlchemy | 2.0 | Web 平台后端 |
| 后端运行时 | Python | 3.11 | Web 平台后端 |
| 后端 Web Server | uvicorn / gunicorn | 0.24 / 21.2 | Web 平台后端 |
| 前端框架 | React | 18 | Web 平台 / 移动套装 / 任务后台 |
| 前端脚手架 | UmiJS Max | 4.x | Web 平台前端 |
| 流程图 | @xyflow/react + elkjs | 12.x / 0.9 | Web 平台前端 |
| 音频波形 | wavesurfer.js | 7.x | Web 平台前端 |
| 桌面壳 | Electron | 33 | 移动套装 |
| 构建工具 | Vite | 6 | 移动套装 / 同步客户端前端 |
| 标注画布 | Konva / react-konva | 10.x / 18.x | 移动套装 |
| 同步语言 | Go | 1.21+ / 1.24+ | 同步服务端 / 客户端 |
| 录音语言 | Go | 1.23 | net_audio |
| 任务后台 | Flask | 3.0 | wxdevops-task-admin |
| 消息缓冲 | Redis Stream | 7.x | 数据同步 |

---

## 第二段 · 逐系统说明

### Web 平台（`smart_tpm_web_v2`）

平台是整套系统的中枢，仓库内含三个子工程：`smart_tpm_edge_api_v2`（后端）、`smart_tpm_edge_web_v2`（前端）、`smart_tpm_sync_api`（同步 API）。

- **后端**：`FastAPI 0.104` + `uvicorn` / `gunicorn`。ORM 用 `SQLAlchemy 2.0`（异步，`aiomysql` / `asyncpg` 双驱动），迁移用 `Alembic`。鉴权是 JWT（`pyjwt` + `passlib[bcrypt]`）。它同时承载 **MCP 端点**（`/api/v2/mcp`），把业务能力暴露给 AI。
- **存储四件套**：`MySQL 8`（业务主库）+ `Redis`（缓存）+ `InfluxDB`（时序指标）+ `MinIO`（S3 兼容对象存储，经 `boto3` 访问，放音频/图片/模型制品）。
- **前端**：`React 18` + `UmiJS Max` 脚手架，UI 用 `Radix UI` + `TailwindCSS`（`class-variance-authority` / `tailwind-merge`）。几个专业能力很关键——
  - `@xyflow/react` + `elkjs`：画检测流程/算法流的**节点图**；
  - `wavesurfer.js`：渲染**音频波形**（声学质检的核心可视化）；
  - `echarts` / `recharts` / `plotly.js`：多种图表；
  - `@tanstack/react-query` + `@tanstack/react-table`：数据请求与表格；
  - 状态用 `zustand`，校验用 `zod`。
- **约定**：仓库有一条硬规则——项目内只允许 `CLAUDE.md` / `README.md`，其他文档写到外部知识库；前端每次提交自动 bump patch 版本号（见仓库 `CLAUDE.md`）。

### 数据同步（`sync-data`）

面向多门店/分支机构：把各源端 MySQL 的变更实时汇聚到中心库，并把关联的 MinIO 文件一起搬过去。分服务端 / 客户端两个 Go 工程。

- **语言**：服务端 `Go 1.21+`，客户端 `Go 1.24+`。
- **同步机制**：客户端监听源库 **Binlog**（`go-mysql`），以**完整事务**为单位组装；经 **gRPC 双向流** 发到服务端；服务端过 `Redis Stream` 缓冲，再 16 分片聚合 → 优先级调度 → 64 Slot 路由 → Worker 写入目标 `MySQL 8` / `PostgreSQL 14+`。
- **关键工程点**：三级反压、断路器、Roaring Bitmap 乱序 checkpoint、CRC32 一致性校验、UPSERT 幂等写入。客户端断点存 `SQLite`(WAL)。
- **前端**：服务端管理台 `React 18 + Ant Design 5`；客户端管理台 `Vue 3 + Vite`。
- **观测**：两端各暴露 32+ 项 `Prometheus` 指标；`Docker Compose` 部署。
- **与本平台的关系**：移动套装/现场采集的数据靠它回中心库，专用同步库 `smart_quality_client_sync`。

### 移动套装（`SmartQuality-client`）

现场用的桌面应用，采集 / 标注 / 上传一体，内部称"移动套装"。

- **壳与语言**：`Electron 33` + `TypeScript`，渲染层 `React 18` + `Vite 6` + `TailwindCSS`。
- **进程结构**：Electron 主进程内还跑一个 `Express` 服务（`server/`），本地充当 API；ORM 用 `TypeORM` + `pg`（连 `PostgreSQL`）。
- **专业能力**：
  - `Konva` / `react-konva`：**标注画布**（画粗标/细标 ROI）；
  - `native/fftw-addon`（cmake-js 编译的 FFTW native addon）：本地做**音频 FFT**；
  - `node-hid`：读硬件设备；`bonjour-service`：**mDNS 发现**局域网内的采集盒子（对接 net_audio）；
  - `jsonwebtoken`：本地鉴权；`winston`：日志；`zustand` / `zod`：状态与校验。
- **发布**：`electron-builder` 打 Windows 包，`electron-updater` 自动更新，`scripts/bump-version.cjs` 管版本。

### 声学算法后端（`ai_api_server_v2` / audio-quality）

检测流程里 `algorithm` 节点最终打到的算力所在。`Python` 写的**自研 DAG 引擎**，跑 21+ 种 `BF_*` 声学算法，双视角聚合出结论；部署在 GPU 集群，存储用 `MySQL` / `MinIO` / `Qdrant`（向量库）。协议是 `POST /detect/execute`。技术栈细节以其自带的技术栈文档为准，见子百科 [about-audio-quality](../../about-audio-quality/SKILL.md)。

### 联网录音（`net_audio` / SmartAudio）

工业现场的联网录音硬件盒子的软件。`Go 1.23` + `PortAudio`（音频采集）双进程架构（采集进程 + 管理进程），管理端 Web 用 `Vue 3` embed，服务发现用 `zeroconf`(mDNS)，进程托管 `systemd`，灵敏度调校涉及 `TPL0501` 数字电位器等硬件。给上游算法提供 `stop_and_fetch` 拉音频接口。细节见子百科 [about-net-audio](../../about-net-audio/SKILL.md)。

### 任务协同后台（`wxdevops-task-admin`）

不是检测链路的一环，而是团队协同工具：监听企业微信群聊、用 AI 识别任务指派、同步到阿里云云效建工作项。

- **后端**：`Python 3.9` + `Flask 3.0` + `Flask-SQLAlchemy` + `PyMySQL`；企业微信会话存档消息解密用 `PyCryptodome`。
- **前端**：`React 18` + `Ant Design 5` + `React Router 6` + `Axios`。
- **AI**：多模型路由（豆包 / Dify / OpenAI / Claude），按配置的默认 provider 分发。
- **存储**：`MySQL`。

### 日报 App 源码（`framework_v3_smart_quality`）

**只放日报平台（内部 Tinia v3）上运行的 app 源码，不含平台本体**。三类 app 各一个顶级目录：`daily/`（日报 endpoint，每次请求跑一次）、`plugins/`（DAG 节点插件，one-shot worker）、`apps/`（常驻服务）。用 `report_sdk` / `bestfunc_sdk`，`Python` 写。依赖走离线 wheelhouse（预装一批锁版本 wheel），版本走 git tag `<slug>/vX.Y.Z`。它与本平台的接点是"把检测数据渲染成日报/看板"。

### MCP 插件仓（`smart_quality_plugins`）

**本仓库自己**。技术栈极简——只有 `Markdown`（skill 文档）+ `JSON`（`plugin.json` / marketplace 清单），遵循 Claude Code plugin 规范。它不含运行时代码，价值在于：定义 MCP connector 指向后端 `/api/v2/mcp`，用 `OAuth 2.1 + PKCE + DCR` 免 token 授权，并携带一批业务 / 百科 skill。工具数量以后端 `tools/list` 为准，不在本文写死。详见 [12-ecosystem.md](./12-ecosystem.md)。

---

### 这些系统怎么拼成一条链路

单看每个仓库是零件，串起来才是产品。一条典型的数据流：

```
现场声音
  → net_audio 采集（Go/PortAudio，A/B/C/D 通道）
  → 移动套装 SmartQuality 客户端 现场录制/标注/上传（Electron/React）
      或 产线实时检测
  → 声学后端 ai_api_server_v2 跑 BF_* 算法（Python/DAG，出结论）
  → Web 平台 记录检测结果 / 编排检测流程（FastAPI + React）
      · 存 MySQL（业务）/ InfluxDB（指标）/ MinIO（音频、制品）
  → sync-data 把现场库变更实时汇聚回中心库（Go/gRPC/Binlog）
  → 日报 App（framework_v3）把数据渲染成看板/日报
  → MCP 插件（smart_quality_plugins）让 AI 用自然语言查这一切
```

每个箭头两端往往是不同语言/不同仓库，靠 REST / gRPC / MCP / 对象存储 松耦合对接。想弄清某一段业务含义，配合看 [04-key-concepts.md](./04-key-concepts.md)。

---

## 第三段 · 横切关注点

跨多个系统复用的基础设施与约定：

| 关注点 | 选型 | 谁在用 |
|---|---|---|
| 关系型主库 | MySQL 8 | Web 平台、任务后台、同步目标库之一 |
| 关系型（客户端/目标） | PostgreSQL 14+ | 移动套装、同步目标库之一 |
| 缓存 / 消息缓冲 | Redis（含 Redis Stream） | Web 平台、数据同步 |
| 时序指标 | InfluxDB | Web 平台 |
| 对象存储 | MinIO（S3 兼容） | Web 平台、数据同步、声学后端 |
| 向量存储 | Qdrant | 声学算法后端 |
| 服务发现 | mDNS / zeroconf / bonjour | net_audio、移动套装 |
| 服务间通信 | gRPC 双向流 | 数据同步 |
| 对外 API | REST / HTTP | 各系统 |
| AI 集成 | MCP over HTTP | Web 平台 ↔ 插件仓 |
| 鉴权 | JWT（各系统内）· OAuth 2.1 + PKCE + DCR（MCP 层）· ApiKey（服务到服务） | 全系统 |
| 观测 | Prometheus（+ 平台侧 Grafana/ELK 可选） | 数据同步、Web 平台 |
| 打包部署 | Docker / Docker Compose | 各服务端 · Electron builder（移动套装） |

几条值得记住的取舍：

- **语言按场景分**：高并发/低延迟的采集与同步用 `Go`；业务逻辑密集、要接 AI/算法生态的用 `Python`；界面与桌面端用 `TypeScript`。
- **对象存储统一 MinIO**：音频、图片、模型制品都走 S3 兼容接口，跨系统一致；文件下载有配套的姊妹插件走同一 MCP 端点。
- **鉴权分三层**：系统内用 JWT；AI 客户端接入用 OAuth（绑用户+组织）；服务到服务中转用 ApiKey（不绑用户）。三者的关系见 [smart-tpm-business-concepts](../../smart-tpm-business-concepts/SKILL.md)。
- **未来演进**：更多算法上向量检索、异步队列、更多 MCP 工具等能力在各仓库的路线图中处于**计划 / 调研**阶段，落地节奏以各仓库自身文档为准。

### 测试与质量工具

| 系统 | 单测 | 端到端 / 其他 |
|---|---|---|
| Web 平台后端 | pytest | locust（压测）· flake8 / mypy / black / isort |
| Web 平台前端 | 自带 test 脚本 | ESLint · type-check · 视觉回归 |
| 移动套装 | Vitest | Playwright（e2e）· ESLint · tsc |
| 数据同步 | Go test | Prometheus 指标断言 |
| 任务后台 | — | 依赖企业微信/云效沙箱联调 |

### 部署形态

| 系统 | 打包 | 编排 |
|---|---|---|
| Web 平台 | Docker 镜像 | Docker Compose（多环境 yml）· 亦支持 K8s |
| 数据同步 | Docker（server / client 各一镜像） | Docker Compose |
| 声学后端 | Docker | GPU 集群 + nginx 分流（见子百科） |
| 联网录音 | 出厂镜像 | systemd 托管（见子百科） |
| 移动套装 | electron-builder（Windows 包） | electron-updater 自动更新 |
| MCP 插件 | 纯文本，无构建 | git 发版 + `_shared` 同步到 3 变体 |

> 平台侧还可选接入 Grafana / Prometheus / ELK 做监控与日志聚合；具体是否启用随环境而定。

### 仓库定位速查

交接时最常问的是"这个仓库到底管什么"，一句话对照：

| 仓库 | 一句话职责 |
|---|---|
| `smart_tpm_web_v2` | 平台中枢：组织建模 + 检测流程编排 + 记录管理 + MCP 端点 |
| `ai_api_server_v2` | 声学算法后端：DAG 引擎跑 `BF_*` 算法出结论 |
| `net_audio` | 现场联网录音硬件盒子的软件 |
| `SmartQuality-client` | 移动套装：现场采集/标注/上传桌面应用 |
| `sync-data` | 多源库变更 + 文件实时汇聚回中心库 |
| `wxdevops-task-admin` | 团队协同：企业微信 → AI 识别 → 云效建任务 |
| `framework_v3_smart_quality` | 日报平台上运行的 app 源码（渲染看板/日报） |
| `smart_quality_plugins` | 把以上能力暴露给 AI 的 MCP 插件市场（本仓库） |

延伸阅读：核心概念见 [04-key-concepts.md](./04-key-concepts.md)；插件与子百科地图见 [12-ecosystem.md](./12-ecosystem.md)；声学后端/联网录音的技术栈以各自子百科为权威——[about-audio-quality](../../about-audio-quality/SKILL.md)、[about-net-audio](../../about-net-audio/SKILL.md)。

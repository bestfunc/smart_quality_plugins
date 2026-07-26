# SmartQuality 整体架构

本文件描述 SmartQuality（Smart TPM）声学质量检测平台的**端到端架构**：一件产品从产线上被录音开始，它的音频、判定结果、标注数据要经过哪些系统、以什么协议流动、最终如何被 AI 消费。若你还不清楚"有哪些系统、各自干什么"，请先读 [`01-product-overview.md`](01-product-overview.md) 再回到本文。

架构上，整套产品是一条**自下而上的数据管道**：现场采集 → 数据同步 → 中心检测平台 → 分析/日报 → AI 插件层。每一层职责单一、通过明确的协议（gRPC / REST / MCP）与上下游解耦，因此可以按层理解、按层交接。本文先给一张总图，再逐层拆解，最后给出跨层的关键约定。

> 阅读约定：涉及未来能力的段落用"计划 / 设想 / 目前倾向"等弱措辞标注；端口号只写通用默认值，不写任何具体内网地址。

---

## 一、端到端架构总图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  第 0 层 · 产线现场（某产线 / 某客户）                                            │
│  设备运转声 → 麦克风 / 采集设备 → PCM 音频                                        │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                 │ 录音
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  第 1 层 · 移动/采集客户端                                                        │
│  ┌────────────────────────────┐        ┌────────────────────────────┐          │
│  │ 移动套装桌面端             │        │ 账号 / 激活服务             │          │
│  │ SmartQuality-client        │◄──────►│ SmartQuality-api            │          │
│  │ Electron+React+Express     │  激活/  │ Flask+React (:5000)         │          │
│  │ 原生 FFTW · 采集+本地检测  │  找回   │ 用户/激活/找回密码          │          │
│  └──────────────┬─────────────┘         └────────────────────────────┘          │
│                 │ 本地 MySQL 数据变更                                             │
└─────────────────┼──────────────────────────────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  第 2 层 · 数据同步（多源 → 中心库）    仓库: sync-data                           │
│  ┌────────────────────────┐   gRPC 双向流    ┌────────────────────────────┐     │
│  │ sync-client（源端）    │ ───────────────► │ sync-server（中心）        │     │
│  │ Binlog 监听 + 全量扫描 │   PCM/文件依赖   │ Redis Stream 缓冲 → 事务   │     │
│  │ SQLite 断点 (Go)       │ ───────────────► │ 聚合 → 写中心库 (Go)       │     │
│  └────────────────────────┘   MinIO 文件同步 └─────────────┬──────────────┘     │
│                                                            │ 事务级写入          │
└────────────────────────────────────────────────────────────┼───────────────────┘
                                                             │
                                                             ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  第 3 层 · Web 检测平台（中心）        仓库: smart_tpm_web_v2                     │
│  React 前端 (:8005) ◄──► FastAPI 后端 (:8000)                                    │
│      检测流程 detect flow · 算法 & 参数 · OK/NG 判定 · 数据集 & 标注 · 生产记录  │
│  ┌────────┐ ┌────────┐ ┌────────────┐ ┌──────────┐                              │
│  │ MySQL  │ │ Redis  │ │ InfluxDB   │ │ MinIO    │                              │
│  │ 业务库 │ │ 缓存   │ │ 时序指标   │ │ 对象存储 │                              │
│  │ :3306  │ │ :6379  │ │ :8086      │ │ 音频/模型│                              │
│  └────────┘ └────────┘ └────────────┘ └──────────┘                              │
└───────────────┬────────────────────────────────────────────┬───────────────────┘
                │ REST                                        │ MCP over OAuth2.1
                ▼                                             ▼
┌───────────────────────────────────────┐   ┌──────────────────────────────────────┐
│  第 4 层 · Tinia 分析 / 日报           │   │  第 5 层 · AI 插件层（MCP）          │
│  Tinia_nodes_smart_quality_app（节点） │   │  smart_quality_plugins（61 tool）    │
│  framework_v3_smart_quality（app 源码）│   │  bestfunc_wechat_plugin（微信只读）  │
│  bi（日报平台 Go+PG，活报告/快照）     │   │  ── 工程师用自然语言查数据 / 追链路 / │
│  数据集节点拉录音 → 分析流 → 日报      │   │     复现报错 / 辅助打标（本百科所在） │
└───────────────────────────────────────┘   └──────────────────────────────────────┘

         旁支 · 企业微信服务（外围协作，弱耦合于检测主链）
         wxdevops-task-admin（监听群聊+AI识别任务→云效; 亦是微信 MCP 服务端 :8090/:5000）
         └── bestfunc_wechat_plugin（Claude Code 只读插件）
```

> 图中端口均为各仓库 README 给出的**通用默认值**（开发态），生产部署可能不同，实际以部署配置为准。

---

## 二、分层职责总表

| 层 | 名称 | 主要仓库 | 职责 | 技术栈 | 与上下游的接口 |
|---|---|---|---|---|---|
| 0 | 产线现场 | —（物理层） | 采集设备运转声，产出 PCM 音频 | 麦克风 / 采集硬件 | 物理录音 → 第 1 层 |
| 1 | 移动/采集客户端 | `SmartQuality-client`、`SmartQuality-api` | 现场录音、本地检测、项目管理；账号激活与找回密码 | Electron+React+Express+FFTW / Flask+React | 本地 MySQL 变更 → 第 2 层；REST 授权 → 第 4 层 |
| 2 | 数据同步 | `sync-data`（server+client） | 把多源 MySQL 增量/全量+关联文件事务级汇聚到中心库 | Go + gRPC + Redis Stream + MySQL/PG + MinIO | Binlog 采集 ← 第 1 层；事务写入 → 第 3 层 |
| 3 | Web 检测平台 | `smart_tpm_web_v2` | 检测流程/算法/OK-NG 判定/数据集/标注/生产检测记录的中心 | FastAPI+React+MySQL+Redis+InfluxDB+MinIO | REST → 第 4 层；MCP → 第 5 层 |
| 4 | Tinia 分析/日报 | `Tinia_nodes_smart_quality_app`、`framework_v3_smart_quality`、`bi` | 把检测数据加工成分析流程与可分享日报（活报告 + 静态快照） | Python 节点 + Tinia SDK / Go+Gin+GORM+PG+React | REST 拉数据 ← 第 3 层 |
| 5 | AI 插件层 | `smart_quality_plugins`、`bestfunc_wechat_plugin` | 通过 MCP 把检测数据/微信群聊以"tool"喂给本地 AI | Claude Code 插件市场 + MCP + OAuth2.1 | MCP over OAuth ← 第 3 层 / 企业微信服务 |

---

## 三、逐层详解

### 第 0 层 · 产线现场

**职责**：这是数据的源头——某产线上的设备在运转、装配或通电时发出的声音，被工位上的麦克风/采集设备录成 **PCM（Pulse Code Modulation，脉冲编码调制）音频**。PCM 是最原始的无压缩数字音频格式，后续各层都围绕它做搬运与分析。

**注意事项（交接坑点）**：PCM 的**字节序（endianness）**需要留意。移动套装侧假设底层存储的 PCM 是 **big-endian（大端）**，接入 Tinia 时需做 byte-swap 转成播放器可播的 WAV；若某些设备录入的是 little-endian（小端），沿用同一套 swap 逻辑会得到错误波形。详见 [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md)。

### 第 1 层 · 移动/采集客户端

**职责**：把第 0 层的录音"落地成业务数据"。这一层有两个仓库：

| 仓库 | 角色 | 关键点 |
|---|---|---|
| `SmartQuality-client` | **移动套装桌面端**（内部称"移动套装"） | Electron 桌面 App（React 前端 + 内嵌 Express 服务端），带**原生 FFTW**（Fastest Fourier Transform in the West，做频谱/FFT 声学分析的 C 库，通过 cmake-js 编译成 Electron 原生插件）；负责现场采集、本地检测、项目/录音管理 |
| `SmartQuality-api` | **账号/激活服务** | Flask + React + Antd 全栈；管理端做用户增删改（JWT 鉴权），客户端侧做**激活（activate）**、发送验证码、重置密码；后台库 `smart_quality`（MySQL 8.0）；默认端口后端 `:5000`、前端 `:80` |

**数据流向**：桌面端在本地 MySQL 记录录音与检测数据 → 这些变更由第 2 层的 sync-client 监听采集。桌面端还通过 REST 向第 4 层的 Tinia 集成节点授权并提供录音下载（见下）。

**技术栈要点**：`SmartQuality-client` 用 Electron 33 + React 18，`electron-builder` 打包，原生插件走 `cmake-js` 针对 Electron runtime 编译。`SmartQuality-api` 部署用 Docker + Gunicorn + Nginx。

详情见 [`../systems/04-app-client.md`](../systems/04-app-client.md) 与 [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md)。

### 第 2 层 · 数据同步（sync-data）

**职责**：这是把"散在多个现场/门店的 MySQL"汇聚成"一个中心库"的管道，也是整套架构里工程复杂度最高的一层。仓库 `sync-data` 内含两端：

**sync-client（源端，装在每个现场）**：
- **Binlog 监听** — Binlog 是 MySQL 记录所有数据变更的二进制日志；客户端用 `go-mysql` 实时读取，把行级变更（增删改）组装成完整**事务**再上送，保证目标库一致性。
- **全量扫描** — 按需对整表做快照扫描，用于首次接入或补数。
- **文件依赖同步** — 数据行若关联 MinIO 中的音频/文件，一并同步。
- **断点续传** — 用本地 **SQLite（WAL 模式）** 记录 checkpoint，断线后从断点继续。
- **双目标连通探测** — 支持探测两个目标地址的连通性做切换。

**sync-server（中心，一套）**：接收端做了完整的流控与可靠性工程——三级**反压（backpressure）**监控、**Redis Stream** 消息缓冲、16 分片事务聚合、优先级调度+令牌桶、64 Slot 路由、MySQL/PG 写入 worker 带**断路器（circuit breaker）**、DDL 串行处理、Roaring Bitmap checkpoint、CRC32 一致性校验（reconciler）、死信队列、以及 32+ 项 Prometheus 指标与 React 管理后台。

| 维度 | sync-client | sync-server |
|---|---|---|
| 语言 | Go 1.24+ | Go 1.21+ |
| 通信 | gRPC 双向流 | gRPC 双向流 |
| 本地存储 | SQLite（断点） | MySQL（元数据） |
| 目标库 | — | MySQL 8.0+ / PostgreSQL 14+ |
| 缓冲 | — | Redis Stream 7.x |
| 前端 | Vue 3 + Vite | React 18 + Ant Design 5 |
| 对象存储 | MinIO | MinIO |

**数据流向**：sync-client 采集 → gRPC 双向流上送 → sync-server 缓冲/聚合/写入中心库（供第 3 层 Web 检测平台读取）。另支持 **NDJSON** 文件的离线导入（三层去重），用于网络隔离现场的离线补数。

详情见 [`../systems/02-data-sync-service.md`](../systems/02-data-sync-service.md)（服务端）与 [`../systems/03-data-sync-client.md`](../systems/03-data-sync-client.md)（客户端）。

### 第 3 层 · Web 检测平台（smart_tpm_web_v2）

**职责**：这是整套产品的**核心中枢**，中心化的检测流程与数据资产都在这里。它承载：

- **检测流程（detect flow）** — 产品从录音到出 OK/NG 结论要经过的算法节点编排；含流程定义、节点、执行链路与日志。
- **算法与参数** — 算法、算法流（algorithm flow）、算法参数库。
- **OK/NG 判定** — 生产线上每个检测通道（detect channel）的检测记录，按工位分库存储。
- **数据集与标注** — 数据集/样本/标注，以及**标记系统**：项目标记字典、粗标（rough）、细标（detail）、标注历史（审计）。
- **元数据** — 产品、设备、装配线、项目、工位、用户、模型 artifact 等反查维度。

**技术栈**：后端 **FastAPI**（Python 3.11+，默认 `:8000`，自带 `/docs` 交互文档），前端 **React**（默认 `:8005`），四件数据存储各司其职：

| 存储 | 默认端口 | 存什么 |
|---|---|---|
| **MySQL** | 3306 | 业务关系数据（流程、算法、数据集、检测记录…） |
| **Redis** | 6379 | 缓存 |
| **InfluxDB**（时序库） | 8086 | 时间序列指标（监控/趋势） |
| **MinIO**（S3 兼容对象存储） | 9000 | 音频文件、模型 artifact 等大文件 |

> 术语：**InfluxDB** 是专门存"随时间变化的数值序列"的时序数据库；**MinIO** 是自建的、兼容 Amazon S3 接口的对象存储，产品里用来放录音 WAV 与模型文件。仓库 README 另提及可选的 Nginx 负载均衡、Grafana/Prometheus 监控与 K8s 部署清单（生产部署方向，属规划/可选项）。

**数据流向**：向上接第 2 层同步进来的中心数据；向第 4 层通过 **REST** 暴露项目/录音等资源；向第 5 层通过 **MCP** 暴露 61 个业务 tool。

详情见 [`../systems/05-web-platform.md`](../systems/05-web-platform.md)。

### 第 4 层 · Tinia 分析 / 日报

**职责**：把第 3 层的检测数据**加工成分析流程与可分享的报告**。**Tinia** 是内部的分析/日报平台（桌面 Dev Studio + Tinia v3 托管运行时）。这一层由三个仓库协作：

| 仓库 | 角色 | 关键点 |
|---|---|---|
| `Tinia_nodes_smart_quality_app` | **移动套装集成节点** | Python 节点插件（`table_prefix=plg_smart_quality_`）；通过 SmartQuality **REST API** 拉项目录音（**不直连其数据库**），作为 Tinia 的数据集节点（`smart_quality_dataset`），支持项目/关键词/日期/OK-NG/产品/工位/设备等多维筛选，可并行下载 WAV 到 Tinia blob |
| `framework_v3_smart_quality` | **日报平台 app 源码仓** | 只放 Tinia v3 上跑的三类 app 源码（`daily/` 日报 endpoint、`plugins/` 节点插件、`apps/` 常驻服务），版本走 git tag `<slug>/vX.Y.Z` |
| `bi` | **日报平台本体** | Go+Gin+GORM+**PostgreSQL** 后端 + React 前端；托管运行内部自编码 Python app（常驻 worker），产出**活报告**（`/a/<slug>` 实时 iframe 分享链接）与**静态快照**（`/a/<slug>/snap/<id>`，用 chromedp 无头浏览器"烤"成永久 HTML）；调度用 robfig/cron，时区固定 Asia/Shanghai |

> 术语：
> - **活报告（live report）** — 每次打开都实时重跑取数的分享页；**静态快照（snapshot）** — 某一时刻烤死的 HTML，永久不变。
> - **endpoint / 常驻服务 / 节点插件** — Tinia v3 上 app 的三种进程模型：endpoint 每次请求跑一次，常驻服务 always-on 由 Supervisor 托管，节点插件是 DAG 流水线里的 one-shot worker。
> - **凭据 alias** — app 连外部系统（如连 Smart TPM）用的凭据别名，命名硬约束为下划线风格（如 `smart_tpm`）。

**数据流向**：通过 REST 从第 3 层 Web 检测平台拉数据 → 在 Tinia 里编排分析流/日报 → 输出报告链接给分析师。集成节点侧的凭证只存 `{host, token}`，账号密码不入库（授权页签发 token 后账密留在浏览器本地）。

**规划提示**：`bi` 的 README 显示其 dev 分支领先生产实例较多，用户体系（登录/RBAC/审计）、发布通道（邮件/Webhook/钉钉）、Redis、K8s 等为**设想中的 v2 方向**，尚未在生产落地，交接时请勿据此对外承诺。

相关详情可参考 [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md)（移动套装 ↔ Tinia 集成部分）。

### 第 5 层 · AI 插件层（MCP）

**职责**：让工程师**用自然语言**直接问检测数据，而不必手写 SQL 或翻页面。核心是 **MCP（Model Context Protocol）**——一种让本地 AI 客户端（Claude Code / Codex / Qwen Code）安全调用后端"工具"的协议。

| 仓库 | 角色 | 关键点 |
|---|---|---|
| `smart_quality_plugins` | **Smart TPM AI 插件市场**（本百科所在仓） | Claude Code 插件 marketplace；把 Web 检测平台 9 个模块、**61 个业务 tool** 暴露给 AI（数据集/标记/检测流程/算法/测试任务/生产检测记录/链路追踪/元数据反查/配置查询）；配套 skill 覆盖 onboarding、业务概念、字段参考、样本抽取、算法链路定位、复现客户报错、AI 辅助打标等 |
| `bestfunc_wechat_plugin` | **微信只读 MCP 插件** | Claude Code 插件，只读访问有权限的微信群聊/人员/附件；服务端在 `wxdevops-task-admin` |

**鉴权模型**：走 **OAuth 2.1 + PKCE + DCR**（动态客户端注册）——浏览器一次同意即授权，不需复制粘贴 token，token 自动 refresh，可在 Smart TPM Web 个人设置里撤销。权限按 **scope**（作用域）切分，粒度到"模块 × 读/写"：如 `mcp:datasets:read`（数据集读）、`mcp:datasets:write`（含 AI 打粗标/细标，写操作需显式授权）、`mcp:detect_flows:read` 等。检测流程与算法**默认不开写 scope**，防止 AI 改坏生产配置；所有写 tool 自动落审计表，记录真实用户 user_id。

**数据流向**：AI 客户端 → MCP over OAuth → Web 检测平台后端的 MCP server → 只读/受控写返回结构化结果进 AI 上下文。文件下载（MinIO bucket / 数据集音频 / 模型 artifact）走配套的姊妹 Files 插件，与业务插件共用同一 MCP 端点、一次授权双生效。

### 旁支 · 企业微信服务（外围）

`wxdevops-task-admin` 监听企业微信群聊消息、用 AI 识别任务指派、自动同步到阿里云**云效**创建工作项，并管理微信用户与云效用户映射；它同时是 `bestfunc_wechat_plugin` 的 **MCP 服务端**（协议层默认 `:8090`）与 **OAuth 授权服务器**（默认 `:5000`）。这条支线偏运营协作，**与声学检测主链路弱耦合**，交接时可相对独立看待。详见 [`../systems/01-wechat-service.md`](../systems/01-wechat-service.md)。

---

## 四、跨层关键约定

以下约定横跨多层，是理解整套系统运转与排障的关键，单独列出：

| 约定 | 内容 | 涉及层 |
|---|---|---|
| **协议分工** | 层间用 gRPC（同步链路）/ REST（Tinia 拉数）/ MCP over OAuth（AI 消费）三种协议，各自解耦 | 2 / 4 / 5 |
| **音频存储** | 录音 PCM/WAV 走 MinIO（S3 兼容）对象存储，业务库只存 URL/元数据 | 1 / 2 / 3 / 4 |
| **字节序** | PCM 默认按 big-endian 处理，接入播放/分析前 byte-swap 成 WAV | 0 / 1 / 4 |
| **标记两层** | 标注分粗标（rough）+ 细标（detail），写操作全程审计（记真实 user_id） | 3 / 5 |
| **时区** | 日报调度固定 Asia/Shanghai | 4 |
| **写操作最小权限** | MCP 默认只读；检测流程/算法不开写；数据集写需显式 scope | 5 |
| **端口** | 本文所有端口均为通用默认值，生产以部署配置为准；具体内网地址不在本百科落库 | 全部 |

---

## 五、部署形态与排障入口

各层的部署方式与"从哪儿开始查问题"差异较大，交接时按下表定位：

| 层 / 系统 | 部署形态 | 健康/入口线索 | 排障起点 |
|---|---|---|---|
| 移动套装桌面端 | Electron 安装包（electron-builder 打包，含自动更新） | 桌面 App 版本号 | App 本地日志目录 `logs/` |
| 账号/激活服务 | Docker（Gunicorn + Nginx） | 后端 `:5000`、前端 `:80` | 后端接口 `/api/*` 返回 |
| 数据同步服务端 | Docker Compose | 32+ Prometheus 指标、React 管理后台 | 反压/断路器指标、死信队列、reconciler 校验 |
| 数据同步客户端 | Go 二进制 / Docker（可选） | HTTP 管理 UI、Prometheus 指标 | SQLite 断点、Binlog 连通、双目标探测 |
| Web 检测平台 | Docker Compose（dev）/ K8s（生产方向） | FastAPI `:8000/docs`、前端 `:8005` | `/docs` 交互接口、MySQL/Redis/InfluxDB/MinIO 连通 |
| Tinia 集成节点 | 灌入 Tinia Dev Studio 项目目录 | 节点在流程编辑器可见 | `test_connection` handler、REST token 是否有效 |
| 日报平台（bi） | Go server + React client + chromedp sidecar | 活报告 `/a/<slug>`、快照 `/a/<slug>/snap/<id>` | worker pool 日志、cron 调度、chromedp WS |
| AI 插件层 | Claude Code 插件市场安装 | `/mcp` 连接状态、OAuth 授权页 | scope 是否授权、`.well-known/oauth-authorization-server` |
| 企业微信服务 | Flask + React；MCP 服务端 + OAuth AS | MCP `:8090`、OAuth `:5000` | 两个端口是否都可达（常见坑：只放通 8090 挡了 5000） |

> **通用排障顺序建议**：先确认层间协议通不通（gRPC / REST / MCP），再看数据源头（同步链路是否在更新中心库），最后才怀疑业务逻辑。多数"数据不对/不更新"的问题根因在第 2 层同步链路或第 1 层采集端，而非展示层。

---

## 六、按需下钻

| 你想深入的层 | 系统详情页 |
|---|---|
| 企业微信服务（监听 / 任务 / MCP 服务端） | [`../systems/01-wechat-service.md`](../systems/01-wechat-service.md) |
| 数据同步服务端（中心库汇聚） | [`../systems/02-data-sync-service.md`](../systems/02-data-sync-service.md) |
| 数据同步客户端（源端 Binlog 采集） | [`../systems/03-data-sync-client.md`](../systems/03-data-sync-client.md) |
| 账号 / 激活服务 | [`../systems/04-app-client.md`](../systems/04-app-client.md) |
| Web 检测平台（核心中枢） | [`../systems/05-web-platform.md`](../systems/05-web-platform.md) |
| 移动套装（桌面采集端 + Tinia 集成） | [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md) |
| 产品全局 / 系统清单 | [`01-product-overview.md`](01-product-overview.md) |

---

## 小结

SmartQuality 的架构是一条**分层解耦的数据管道**：现场录音（第 0 层）→ 移动/采集客户端落地（第 1 层）→ 数据同步汇聚中心（第 2 层）→ Web 检测平台做判定与数据资产管理（第 3 层）→ Tinia 加工成分析与日报（第 4 层）→ AI 插件层让工程师自然语言消费（第 5 层）。层间靠 gRPC / REST / MCP 明确分工，音频统一走 MinIO，标注与写操作全程审计。掌握这张分层图与跨层约定，再配合各系统详情页，即可支撑离职交接场景下的定位与排障。

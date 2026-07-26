# SmartQuality 产品总览

SmartQuality（内部亦称 **Smart TPM**，TPM = Total Productive Maintenance，全员生产维护）是一套面向工业产线的**声学/音频质量检测平台**。它的核心心智很朴素：把产线上一台台设备运转、装配、通电时发出的**声音录下来**，交给算法判定这台产品是 **OK（合格）还是 NG（No Good，不合格）**，再把这些录音、判定结果、人工复核标注沉淀成**数据集**，供算法迭代与 AI 辅助分析使用。

围绕"录音 → 判定 → 标注 → 数据中心化 → AI 辅助"这条主线，逐步长出了一整套系统：现场采集端、数据同步链路、中心检测平台、分析/日报层、以及把检测数据暴露给本地 AI 的插件层。本文件是这套产品套装的**顶层地图**，帮助刚接手的同事快速建立"有哪些系统、各自干什么、彼此怎么串"的整体认知，再按需下钻到各系统详情页。

> 阅读顺序建议：先读本文建立全局 → 读 [`02-architecture.md`](02-architecture.md) 看端到端数据流 → 按兴趣点进 `../systems/` 下的单系统详情。

---

## 一、解决什么问题

传统产线的声学/听感质检长期依赖**老师傅用耳朵听**：主观、不可追溯、难以规模化、人一走经验就断档。SmartQuality 要把这件事**数字化、算法化、可追溯化**。它把整个质检闭环拆成五个环节：

| 环节 | 做什么 | 通俗解释 |
|---|---|---|
| **1. 产线录音采集** | 在工位上用麦克风/采集设备录下产品运转声（PCM 音频） | 把"老师傅的耳朵"换成麦克风 |
| **2. 算法判 OK/NG** | 录音进检测流程（detect flow），逐级过算法节点，输出合格/不合格 | 把"老师傅的判断"换成算法链路 |
| **3. 数据集 / 标注** | 判定结果+录音归档成数据集，人工复核打"标记"（粗标/细标）纠偏 | 给算法准备"教材"和"错题本" |
| **4. 数据中心化** | 多产线/多门店的数据实时同步汇聚到中心库 | 把散在各现场的数据收拢到一处 |
| **5. AI 辅助** | 通过 MCP 把检测数据暴露给本地 AI，用自然语言查数据、追溯算法链路、复现报错、辅助打标 | 让工程师"用聊天的方式"查数据、排障 |

> **术语先解释**（后文频繁出现）：
> - **OK/NG** — 质检结论，OK = 合格，NG = No Good = 不合格。
> - **detect flow（检测流程）** — 一条产品从录音到出结论要经过的算法节点编排图，是平台的核心配置对象。
> - **数据集（dataset）/ 标记（mark）** — 数据集是归档的样本集合；标记是人工或 AI 对样本贴的判定/属性标签，分**粗标（rough）**与**细标（detail）**两层。
> - **MCP（Model Context Protocol）** — 一种让本地 AI 客户端（Claude Code / Codex 等）安全调用后端能力的协议；本产品用它把检测数据以"工具（tool）"的形式喂给 AI。

这套闭环带来的价值：判定标准统一、每一次判定可回溯到具体算法版本与参数、数据可持续训练模型、且经验沉淀在系统而非个人。

### 为什么用"声音"判质量

很多机械/电子产品的缺陷会先体现在**声学特征**上——异响、啸叫、周期性杂音、频谱异常，往往比外观或电气参数更早暴露装配或部件问题。用声音做质检的难点在于：声音是连续时序信号，人耳判断主观且难量化。SmartQuality 的思路是把音频转成可计算的特征（例如做 **FFT** 频谱分析），再用编排好的算法节点逐级判定，从而把"听感"变成"可复现、可回溯、可训练"的工程量。

因此整套产品的技术重心有三块，交接时可据此理解各系统为什么这样设计：

| 技术重心 | 要解决的核心难题 | 主要承载系统 |
|---|---|---|
| **可靠采集与搬运** | 现场网络不稳、数据不能丢、要能补数 | 移动套装、数据同步链路 |
| **可回溯的判定** | 每条 OK/NG 都要能查到走了哪条算法链、什么参数 | Web 检测平台（检测流程 + 链路追踪） |
| **可持续的数据资产** | 样本要能沉淀、复核、再训练 | 数据集 + 标记系统 + AI 辅助打标 |

---

## 二、产品由哪些系统组成

整套产品是**多仓库协作**的。下表是总索引：每个系统对应一个独立 Git 仓库（部分仓库内含服务端+客户端双端），"交接模块"列指向本百科 `../systems/` 下的详情页，方便离职交接时逐系统对照。

| # | 系统（中文名） | 仓库 | 一句话作用 | 主要技术栈 | 交接模块 |
|---|---|---|---|---|---|
| 1 | **Web 检测平台** | `smart_tpm_web_v2` | 产线录音检测中枢：检测流程/算法/OK-NG 判定/数据集与标注/生产检测记录 | FastAPI + React + MySQL + Redis + InfluxDB + MinIO | [`../systems/05-web-platform.md`](../systems/05-web-platform.md) |
| 2 | **账号/激活服务** | `SmartQuality-api` | 桌面客户端的用户管理、客户端激活、找回密码 | Flask + React + Antd + MySQL | [`../systems/04-app-client.md`](../systems/04-app-client.md) |
| 3 | **移动套装桌面端** | `SmartQuality-client` | 音频质量检测桌面 App：现场采集录音 + 本地检测 + 项目管理 | Electron + React + Node/Express + 原生 FFTW | [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md) |
| 4 | **数据同步服务端** | `sync-data`（`sync-server`） | 汇聚多源 MySQL Binlog，事务级写入中心库 | Go + gRPC + Redis Stream + MySQL/PostgreSQL + MinIO | [`../systems/02-data-sync-service.md`](../systems/02-data-sync-service.md) |
| 5 | **数据同步客户端** | `sync-data`（`sync-client`） | 在源端监听 Binlog 与全量扫描，发往同步服务端 | Go + gRPC + SQLite + go-mysql | [`../systems/03-data-sync-client.md`](../systems/03-data-sync-client.md) |
| 6 | **Tinia 移动套装集成节点** | `Tinia_nodes_smart_quality_app` | 把移动套装的项目录音接入 Tinia 分析流程，作为数据集节点 | Python 节点插件（Tinia 平台） | [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md) |
| 7 | **日报平台 App 源码** | `framework_v3_smart_quality` | Tinia v3 上的三类 app（日报 / 节点插件 / 常驻服务）源码仓 | Python app + Tinia SDK | 见 [`02-architecture.md`](02-architecture.md) 分析/日报层 |
| 8 | **日报平台（Tinia v3）** | `bi` | 托管运行内部自编码 app，产出"活报告"分享链接与静态快照 | Go + Gin + GORM + PostgreSQL + React | 见 [`02-architecture.md`](02-architecture.md) 分析/日报层 |
| 9 | **企业微信任务管理** | `wxdevops-task-admin` | 监听企业微信群聊 → AI 识别任务 → 同步云效；亦是微信只读 MCP 的服务端 | Flask + React + MySQL | [`../systems/01-wechat-service.md`](../systems/01-wechat-service.md) |
| 10 | **微信只读 MCP 插件** | `bestfunc_wechat_plugin` | Claude Code 插件包：只读访问有权限的微信群聊/人员/附件 | Claude Code 插件 + MCP connector | [`../systems/01-wechat-service.md`](../systems/01-wechat-service.md) |
| 11 | **Smart TPM AI 插件市场** | `smart_quality_plugins` | 通过 MCP 把 Web 检测平台的业务能力暴露给本地 AI（61 个 tool + 多个 skill）；**本百科所在仓** | Claude Code 插件市场 + MCP | 见 [`02-architecture.md`](02-architecture.md) AI 插件层 |

> 说明：
> - `sync-data` 一个仓库内含 `sync-server`（服务端）与 `sync-client`（客户端）两端，故拆成两行（#4、#5），分别对应两个交接模块。
> - `wxdevops-task-admin`（#9）同时是 `bestfunc_wechat_plugin`（#10）的 MCP 服务端与 OAuth 授权服务器，两者共同构成"企业微信服务"这一交接模块。
> - `framework_v3_smart_quality`（#7）与 `bi`（#8）共同构成"日报/分析层"：`bi` 是平台本体，`framework_v3_smart_quality` 是跑在其上的 app 源码。
> - 企业微信任务管理（#9/#10）相对偏运营协作，是产品套装的外围支撑系统，与声学检测主链路耦合较弱。

---

## 三、系统之间怎么串（一句话版）

详细数据流见 [`02-architecture.md`](02-architecture.md)，这里先给一个"极简因果链"，便于建立直觉：

```
移动套装桌面端 / 产线现场          →  录音采集、本地/平台检测判 OK-NG
   ↓（源端 MySQL 变更）
数据同步客户端 (sync-client)       →  监听 Binlog
   ↓（gRPC 双向流）
数据同步服务端 (sync-server)       →  事务级聚合，写中心库 + 同步 MinIO 文件
   ↓
Web 检测平台 (smart_tpm_web_v2)    →  中心化的检测流程/数据集/标注/生产记录
   ↓（REST / MCP）
Tinia 分析·日报层                   →  数据集节点、日报活报告与快照
   ↓（MCP over OAuth）
AI 插件层 (smart_quality_plugins)  →  工程师用自然语言查数据 / 追链路 / 复现报错 / 辅助打标
```

> **Tinia** 是内部的分析/日报平台（桌面 Dev Studio + Tinia v3 托管），负责把检测数据加工成分析流程与可分享的报告，本身不产生原始录音。

### 一个具体例子（跟着一件产品走一遍）

为了让"串联"落到实处，设想某产线上一件产品从下线到被 AI 追溯的全过程：

1. 产品在工位通电运转，麦克风录下一段 PCM 音频（**第 0 层**）。
2. 移动套装桌面端把这段录音关联到某项目/工位/产品，本地检测流程判定为 **NG**，数据写入桌面端本地 MySQL（**第 1 层**）。
3. sync-client 监听到这条 Binlog 变更，连同录音文件一起通过 gRPC 上送；sync-server 事务级写入中心库，录音落 MinIO（**第 2 层**）。
4. 质量工程师在 Web 检测平台看到这条 NG 生产检测记录，点开算法链路，发现是某算法节点参数偏严；他把该样本收进数据集，人工复核后打上细标纠偏（**第 3 层**）。
5. 分析师的 Tinia 日报里，这条 NG 计入当日不良率趋势；数据集节点也把这段录音拉进分析流用于算法回归（**第 4 层**）。
6. 另一位工程师用 Claude Code 问"某项目今天有哪些 NG、分别走了哪条算法链"，AI 通过 MCP 调 `detect_records_trace` 等 tool 一键拿到三层链路（**第 5 层**）。

这条例子串起了全部六层，也解释了为什么"Web 检测平台是枢纽"——第 4/5 层看到的一切，本质都来自它中心化后的数据。

### 数据形态一览

同一份"录音"在各层以不同形态存在，理解这点有助于排障时判断"数据卡在哪一层"：

| 层 | 数据形态 | 存储位置 |
|---|---|---|
| 第 0 层 | 原始声波 | 物理世界 |
| 第 1 层 | PCM 音频 + 本地业务记录 | 桌面端本地 MySQL / 本地文件 |
| 第 2 层 | 事务化的行变更 + 文件 | Redis Stream 缓冲 → 中心库 + MinIO |
| 第 3 层 | 检测记录 / 数据集样本 / 标注 | 中心 MySQL + InfluxDB + MinIO |
| 第 4 层 | 分析流数据集 / 日报活报告 / 快照 | Tinia blob / bi 的 PostgreSQL |
| 第 5 层 | MCP 工具返回的结构化 JSON | AI 上下文（不落库） |

---

## 四、典型用户角色

不同角色关心系统的不同切面，交接时可据此判断"某人主要接触哪几个仓库"：

| 角色 | 主要诉求 | 主要接触的系统 |
|---|---|---|
| **产线操作员 / 质检员** | 现场录音、看单件 OK/NG、处理 NG 件 | 移动套装桌面端；Web 检测平台（生产检测记录） |
| **质量 / 算法工程师** | 配检测流程、调算法参数、复核标注、追溯某条记录走了哪条算法链 | Web 检测平台（检测流程/算法/数据集/标记）；AI 插件层 |
| **数据 / BI 分析师** | 看趋势、出日报、做统计口径 | 日报平台（`bi` + `framework_v3_smart_quality`）；Tinia 分析流 |
| **运维 / 交接工程师** | 部署、同步链路健康、库/对象存储运维 | 数据同步服务端/客户端；各系统 Docker 部署 |
| **项目经理 / 协作** | 任务分派、进度跟踪、群聊留痕 | 企业微信任务管理（`wxdevops-task-admin`） |
| **AI 客户端使用者** | 用 Claude Code / Codex 自然语言查检测数据、复现客户报错 | AI 插件层（`smart_quality_plugins` + `bestfunc_wechat_plugin`） |

> 现场客户与产线名称在本百科中一律以"**某产线 / 某客户**"泛指，不落具体名称。

---

## 五、命名与别名对照（避免交接踩坑）

同一样东西在不同仓库/文档里常有多个叫法，新接手最容易被名字绕晕。下表把常见别名归一：

| 你可能听到的叫法 | 指的是 | 说明 |
|---|---|---|
| **Smart TPM** / **SmartQuality** / **智能质检** | 整个产品套装 | 同一产品的不同内部叫法；TPM 是历史遗留的项目代号 |
| **Web 检测平台** / **检测平台** / `smart_tpm_web_v2` | 中心检测平台 | 仓库目录里另有 `smart_tpm_edge_api_v2` / `smart_tpm_edge_web_v2` 前后端子目录 |
| **移动套装** / **桌面端** / `SmartQuality-client` | 现场采集桌面 App | "移动套装"是产品术语，实际是 Electron 桌面应用，非手机 App |
| **同步服务 / 中心库汇聚** / `sync-server` | 数据同步服务端 | 与 `sync-client` 同在 `sync-data` 仓 |
| **Tinia** / **日报平台** / **分析平台** | 分析/日报层 | `bi` 是平台本体，`framework_v3_smart_quality` 是其上 app 源码 |
| **MCP 插件** / **AI 插件** / `smart_quality_plugins` | AI 插件层 | 暴露检测数据给本地 AI；本百科就住在这个仓 |
| **企业微信服务** / **云效同步** / `wxdevops-task-admin` | 外围协作系统 | 与检测主链弱耦合 |

> **重点澄清**：产品术语"移动套装"容易让人以为是手机/小程序，实际它是**桌面 App**（Electron）。真正涉及 MCP、日报的"移动/小程序心智"，指的是开发范式（AI 写代码优先、小而快），不是终端形态。

---

## 六、系统间依赖关系（谁离不开谁）

交接时判断"改动 A 会不会影响 B"的速查表。箭头表示"依赖 / 数据来源"方向：

| 系统 | 依赖谁（上游） | 谁依赖它（下游） | 耦合强度 |
|---|---|---|---|
| 移动套装桌面端 | 账号/激活服务（激活鉴权） | 数据同步客户端、Tinia 集成节点 | 强（数据源头之一） |
| 数据同步客户端 | 源端 MySQL Binlog | 数据同步服务端 | 强 |
| 数据同步服务端 | 数据同步客户端、MinIO | Web 检测平台（中心库） | 强 |
| Web 检测平台 | 数据同步服务端汇聚的中心库 | Tinia 分析层、AI 插件层 | **枢纽**（几乎人人依赖） |
| Tinia 分析/日报 | Web 检测平台 REST | 分析师/看报告的人 | 中（只读消费） |
| AI 插件层 | Web 检测平台 MCP 端点 | 用 AI 的工程师 | 中（只读为主、受控写） |
| 企业微信服务 | 企业微信 API、云效 | 微信 MCP 插件 | 弱（外围） |

> 一句话风险提示：**Web 检测平台是枢纽**，改它的数据模型/接口牵一发动全身；数据同步链路是"生命线"，它断了中心库就停止更新；其余层多为下游消费方，改动影响面相对可控。

---

## 七、与本百科其他文件的导航关系

本百科（`about-smart-quality` skill）按"事实层 / 系统层 / 检索层"分目录组织。新接手请按下表取用：

| 你想知道 | 去哪个文件 |
|---|---|
| 产品是什么、有哪些系统（本文） | `reference/01-product-overview.md` |
| 端到端架构、数据流、分层与技术栈 | [`reference/02-architecture.md`](02-architecture.md) |
| 企业微信服务（监听/任务/MCP 服务端） | [`../systems/01-wechat-service.md`](../systems/01-wechat-service.md) |
| 数据同步服务端（中心库汇聚） | [`../systems/02-data-sync-service.md`](../systems/02-data-sync-service.md) |
| 数据同步客户端（源端采集） | [`../systems/03-data-sync-client.md`](../systems/03-data-sync-client.md) |
| 账号/激活服务（桌面端后台） | [`../systems/04-app-client.md`](../systems/04-app-client.md) |
| Web 检测平台（核心中枢） | [`../systems/05-web-platform.md`](../systems/05-web-platform.md) |
| 移动套装（桌面采集端 + Tinia 集成） | [`../systems/06-mobile-suite.md`](../systems/06-mobile-suite.md) |
| 术语速查（OK/NG、detect flow、mark…） | `../glossary/`（词条持续补充中） |
| 常见问题 / 排障 | `../faq/` |
| 端到端操作场景（如"复现某客户报错"） | `../scenarios/` |

> **未来规划弱措辞提示**：本文件只描述当前已由代码/README 证实的事实。产品仍在演进（例如日报平台 `bi` 的用户体系、发布通道等为设想中的 v2 方向），涉及未来能力的段落会以"计划 / 设想 / 目前倾向"等弱措辞标注，请勿据此对外承诺时间点或功能范围。

---

> **本文件的定位**：它是事实层的"总目录"，只回答"是什么、有哪些、怎么串"，不展开单系统的部署与接口细节——那些在 `../systems/` 下。若你发现某系统的仓库结构或职责与本文描述不符（例如新增/合并了仓库），说明产品发生了演进，请同步更新本表与依赖关系图，保持这张"地图"与实际一致。

---

## 小结

SmartQuality 是"把产线听感质检数字化"的一整套系统：**移动套装/现场采集**负责录音与初步判定，**数据同步链路**把多现场数据汇聚到中心，**Web 检测平台**是检测流程与数据集/标注的中枢，**Tinia 分析·日报层**把数据加工成报告，**AI 插件层**让工程师用自然语言直接问数据。理解了这条主链，再配合各系统详情页与架构文档，就能在离职交接场景下快速定位到具体仓库与模块。下一步请阅读 [`02-architecture.md`](02-architecture.md)。

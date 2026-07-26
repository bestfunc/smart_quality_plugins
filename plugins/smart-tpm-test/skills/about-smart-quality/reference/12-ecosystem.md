# 生态 — 插件市场与子百科地图

> SmartQuality（Smart TPM）平台把自己的能力交给 AI 的方式，是一个 **Claude Code 插件 marketplace**：仓库 `smart_quality_plugins`。它不只是"一个插件"，而是一套"市场 + 多环境变体 + 一整排 skill"的组织结构。本文讲清三件事：这个市场长什么样、里面有哪些 skill、以及**本百科（about-smart-quality）与其他子百科怎么分工**——因为本 skill 定位是"产品全貌 / 离职交接总入口"，具体细分层由各子百科负责。

本文分三段：
1. **市场结构** —— marketplace、`_shared`、3 个变体、姊妹插件。
2. **子百科地图** —— 本百科 vs 各子百科的分工，以及全部 skill 清单。
3. **相对路径导航** —— 一张"要查 X 就去 Y"的跳转表。

本文不写内网地址；变体的 URL 以仓库 `README.md` 与各变体配置为准。

---

## 第一段 · 插件市场结构

### 这是一个 marketplace，不是单个插件

`smart_quality_plugins` 在 `.claude-plugin/` 里声明自己是一个 **marketplace**（市场），下面挂着按部署环境切分的多个 plugin。用户按自己所处环境装其中**一个**即可：

```
/plugin marketplace add bestfunc/smart_quality_plugins
/plugin install smart-tpm-<变体>@smart_quality_plugins
```

### 一份原件、三个变体：`_shared` + local/test/prod

仓库 `plugins/` 目录的物理布局：

```
smart_quality_plugins/
├── .claude-plugin/            ← marketplace 清单（列出 3 个 plugin）
└── plugins/
    ├── _shared/skills/        ← skill 的【唯一原件】
    ├── smart-tpm-local/skills/  ← 本地开发变体（localhost 容器）
    ├── smart-tpm-test/skills/   ← 内网测试变体（部门内测）
    └── smart-tpm-prod/skills/   ← 生产变体（内网直连）
```

- **`_shared/skills/`** 是所有 skill 的**唯一事实来源**（single source of truth）。写 skill、改 skill 都改这里。
- **三个变体**（local / test / prod）各自持有一份 skill 的**物理副本**——发版时从 `_shared` 复制过去，**不用 git symlink**（symlink 在 Windows 上不稳）。三者差别只在指向的后端 URL / 环境，skill 内容一致。
- **切环境**：改 `.claude/settings.json` 里 `enabledPlugins` 的变体后缀（`-local` ↔ `-test` ↔ `-prod`）再重启客户端。

| 变体 | 定位 | 后端 |
|---|---|---|
| `smart-tpm-local` | 本地开发 | 本机 docker 起的后端容器 |
| `smart-tpm-test` | 内网测试 | 部门内测环境 |
| `smart-tpm-prod` | 生产 | 部门生产环境（内网直连，避开自签证书） |

> 具体 URL 见仓库 `README.md`；本文按环境名指代，不写地址。

**三选一怎么选**：本机在跑后端容器调试 → `local`；连部门内测环境验证 → `test`；查真实生产数据 → `prod`。同一时间只装一个变体即可；换环境改后缀重启，不用重装。三者 skill 内容完全一致，差别只在 MCP connector 指向的后端。

### 插件里装了什么

每个变体插件 = **一个 MCP connector + 一整排 skill**：

- **MCP connector**：指向后端 `/api/v2/mcp` 端点，用 **OAuth 2.1 + PKCE + DCR** 免 token 授权（浏览器一次同意）。它把后端多个业务模块的工具暴露给 AI；工具数量与清单**以后端 `tools/list` 为准**，不在文档写死。
- **skill 集**：`_shared/skills/` 下的全部 skill（见第二段清单）。

### marketplace 清单长什么样

`.claude-plugin/` 里的清单声明市场名、owner、version，以及 `plugins[]` 数组——每个元素是一个变体，含 `name` / `source`（指向 `./plugins/smart-tpm-<变体>`）/ `description`。市场整体版本与 `CHANGELOG.md` 对齐，发一次版三个变体一起动。

### 姊妹插件：文件下载

业务工具走本市场；**文件下载**（MinIO bucket / 数据集音频 / 模型制品）走配套的姊妹插件 `SmartTPM_Files_Plugin`。两个插件**指向同一个 MCP server 端点**，OAuth 一次浏览器同意即可两个都生效。分工：本市场管"查数据、追链路、打标记"，姊妹插件管"把文件拉下来"。

### 相邻但不属于本市场的生态

新人容易把几个"沾边"的东西和本市场混起来，先划清边界：

| 名字 | 是什么 | 与本市场的关系 |
|---|---|---|
| `SmartTPM_Files_Plugin` | 文件下载姊妹插件市场 | 同一 MCP 端点，配套使用；管文件不管业务查询 |
| 日报平台（内部 Tinia v3） | 跑日报/看板 app 的独立平台 | 消费本平台数据，但**不是**本市场；app 源码在 `framework_v3_smart_quality` |
| 各产品仓库自身的 skill | 如声学后端仓里的 `add-endpoint` / `debug-assistant` 等开发类 skill | 属于各自仓库，不在本市场；本市场只放"查/追/标 + 百科" |

一句话：**本市场只装"给 AI 查检测平台数据 + 产品百科"这一类 skill**，具体某个产品仓库的开发工具留在那个仓库里。

### 授权与 scope（速览，细节转子百科）

MCP connector 用 **OAuth 2.1 + PKCE + DCR** 授权：无需复制粘贴 token，浏览器一次同意，token 自动刷新，可在平台侧撤销。权限按"模块 × 读写"切成若干 **scope**（如数据集读 / 数据集写 / 检测流程读 / 测试任务读写 / 算法读 / 生产检测记录读等）；写 scope 独立开、默认更谨慎（防 AI 改坏生产配置）。scope 的完整清单以后端 `.well-known/oauth-authorization-server` 的 `scopes_supported` 为准，细节讲解转子百科 [about-smart-tpm-mcp](../../about-smart-tpm-mcp/SKILL.md)。

---

## 第二段 · 子百科地图与 skill 清单

### 本百科（about-smart-quality）的定位

`_shared/skills/` 下有多个以 `about-*` 命名的**百科型 skill**（user-invocable，纯文档、不调工具）。它们构成一张"分层的知识网"，**本百科站在最顶层**：

- **本百科 = 产品全貌 + 离职交接总入口**。回答"这整套平台是什么、有哪些系统、谁负责什么、新人从哪看起、离职要交接哪些"这类**跨系统、总览性**问题。它不深入任何单一子系统的实现细节。
- **各子百科 = 细分层专家**。每个只讲自己那一块（MCP 集成层、声学算法后端、联网录音系统……），深到字段、协议、踩坑。

一句话分工：**本百科给你地图，子百科给你放大镜。** 被问到细节时，本百科负责把人**指路**到对的子百科，而不是自己复述细节。

### 百科型 skill（子百科）

| 子百科 skill | 讲什么细分层 | 路径 |
|---|---|---|
| `about-smart-quality`（本篇） | 产品全貌 / 系统清单 / 交接总入口 | 就是本目录 |
| `about-smart-tpm-mcp` | MCP 集成层：marketplace、变体、OAuth、scope、tool、skill 导览 | [../../about-smart-tpm-mcp/SKILL.md](../../about-smart-tpm-mcp/SKILL.md) |
| `about-audio-quality` | 声学算法后端 `ai_api_server_v2`：DAG 引擎、`BF_*` 算法、GPU 集群 | [../../about-audio-quality/SKILL.md](../../about-audio-quality/SKILL.md) |
| `about-net-audio` | 联网录音 `net_audio`/SmartAudio：双进程、A/B/C/D 通道、灵敏度、部署 | [../../about-net-audio/SKILL.md](../../about-net-audio/SKILL.md) |

### 业务 skill（速查 / 任务）

除了百科，`_shared/skills/` 还有一批**业务 skill**——有的是纯文档速查（Reference），有的会实际调 MCP 工具干活（Task）：

| skill | 类型 | 干什么 | 路径 |
|---|---|---|---|
| `quickstart-smart-tpm-mcp` | Onboarding | 第一次接入：装插件 → 授权 → 跑通第一个查询 | [../../quickstart-smart-tpm-mcp/SKILL.md](../../quickstart-smart-tpm-mcp/SKILL.md) |
| `smart-tpm-business-concepts` | Reference | 业务实体关系权威版（项目/工位/流程/任务…） | [../../smart-tpm-business-concepts/SKILL.md](../../smart-tpm-business-concepts/SKILL.md) |
| `dataset-fields-reference` | Reference | 数据集字段、`markCode`、`sampleRatio` 细节 | [../../dataset-fields-reference/SKILL.md](../../dataset-fields-reference/SKILL.md) |
| `detect-flow-node-reference` | Reference | 检测流程节点字段（merge 三开关等） | [../../detect-flow-node-reference/SKILL.md](../../detect-flow-node-reference/SKILL.md) |
| `sample-dataset-extraction` | Task | 抽取数据集样本与标注、导出 manifest | [../../sample-dataset-extraction/SKILL.md](../../sample-dataset-extraction/SKILL.md) |
| `locate-test-record-algorithm-chain` | Task | 从测试记录定位到算法链路 | [../../locate-test-record-algorithm-chain/SKILL.md](../../locate-test-record-algorithm-chain/SKILL.md) |
| `create-test-task-from-template` | Task（写） | 基于历史任务模板新建测试任务草稿 | [../../create-test-task-from-template/SKILL.md](../../create-test-task-from-template/SKILL.md) |
| `reproduce-customer-error` | Task | 复现现场报错：链路追踪 + 标注历史 + 算法调用快照 | [../../reproduce-customer-error/SKILL.md](../../reproduce-customer-error/SKILL.md) |
| `query-detect-records` | Task | 查生产检测记录（工位分库）+ 文件下载 | [../../query-detect-records/SKILL.md](../../query-detect-records/SKILL.md) |
| `mark-samples-with-ai` | Task（写） | AI 打粗标 / 细标（需数据集写权限） | [../../mark-samples-with-ai/SKILL.md](../../mark-samples-with-ai/SKILL.md) |

> 上表的 skill 名称与类型以仓库 `README.md` 和各 `SKILL.md` frontmatter 为准；数量会随发版增减。"写"类 skill 依赖对应的 write scope，且写操作自动留审计（见 [04-key-concepts.md](./04-key-concepts.md) 的标注审计段）。

### skill 的骨架长什么样

百科型 skill（含本篇）沿用一套统一目录骨架，方便读者按"深度"取用：

| 层 | 目录 | 放什么 |
|---|---|---|
| 事实层 | `reference/` | 编号 01–12 的事实文档（本文即 `reference/12-ecosystem.md`） |
| 场景层 | `scenarios/` | 端到端流程（新人上手 / 交接 / 排障走一遍） |
| 话术层 | `faq/` | 高频问答；带 `*(内部参考)*` 标记的不对外 |
| 术语层 | `glossary/` | 专有名词与缩写表 |

业务型 skill 通常只有一个 `SKILL.md`（Reference/Task），不铺这套骨架。市场里还有一份 `generate-product-encyclopedia.md` 作为"如何写一本产品百科"的模板参考。

### 发版与同步规则

- **改哪里**：所有 skill 只改 `_shared/skills/` 下的原件。
- **怎么分发**：发版时把改动**物理复制**到 `smart-tpm-{local,test,prod}/skills/` 三个变体目录（不用 symlink）。
- **版本**：bump marketplace 清单的 `version` + 在 `CHANGELOG.md` 顶部加一行，git commit + tag。
- **冲突以谁为准**：本文与 `README.md` / `CHANGELOG.md` / marketplace 清单不一致时，以后三者为准。

### 百科之间怎么协作（举例）

- 有人问"这平台整体架构 / 我新来该看什么" → 本百科负责答，并按 systems/ 与 scenarios/ 层分流。
- 追问"MCP 授权失败 / scope 不够 / 变体怎么切" → 指到 [about-smart-tpm-mcp](../../about-smart-tpm-mcp/SKILL.md)。
- 追问"声学算法为什么判 NG / 某个 `BF_*` 算法干嘛" → 指到 [about-audio-quality](../../about-audio-quality/SKILL.md)。
- 追问"现场盒子 mDNS 找不到 / A/B/C/D 通道怎么映射" → 指到 [about-net-audio](../../about-net-audio/SKILL.md)。
- 追问"某个业务术语到底啥意思" → 先给本百科 [04-key-concepts.md](./04-key-concepts.md)，需要字段级再转对应 Reference skill。

---

## 第三段 · 相对路径导航速查

从**本文件（`reference/12-ecosystem.md`）**出发的跳转坐标：

| 想找 | 去哪 |
|---|---|
| 本百科的核心概念 | [./04-key-concepts.md](./04-key-concepts.md) |
| 本百科的技术栈 | [./09-tech-stack.md](./09-tech-stack.md) |
| 本百科的系统分册 | `../systems/`（各系统一册） |
| 本百科的场景/交接流程 | `../scenarios/` |
| 本百科的术语表 | `../glossary/terms.md` |
| MCP 集成层子百科 | [../../about-smart-tpm-mcp/SKILL.md](../../about-smart-tpm-mcp/SKILL.md) |
| 声学算法后端子百科 | [../../about-audio-quality/SKILL.md](../../about-audio-quality/SKILL.md) |
| 联网录音子百科 | [../../about-net-audio/SKILL.md](../../about-net-audio/SKILL.md) |
| 业务概念权威版 | [../../smart-tpm-business-concepts/SKILL.md](../../smart-tpm-business-concepts/SKILL.md) |
| 数据集字段速查 | [../../dataset-fields-reference/SKILL.md](../../dataset-fields-reference/SKILL.md) |
| 检测流程节点速查 | [../../detect-flow-node-reference/SKILL.md](../../detect-flow-node-reference/SKILL.md) |
| 市场仓库总说明 | 仓库根 `README.md` |

**路径小抄**：本文件在 `about-smart-quality/reference/` 下。`./` 指同目录的兄弟文件；`../` 回到 `about-smart-quality/`（下有 systems / scenarios / faq / glossary / reference）；`../../` 回到 `_shared/skills/`，其下平铺着所有兄弟 skill（含全部子百科与业务 skill）。所有子百科与本百科**同级**，都在 `_shared/skills/` 下。

### 当你被问到的时候（分流示例）

> **新人**：这套东西一共有哪些系统、我该从哪看起？
> **你**：给本百科的 systems/ 分册总览 + [04-key-concepts.md](./04-key-concepts.md) + [09-tech-stack.md](./09-tech-stack.md)，先建立全貌再钻子系统。

> **要交接的人**：离职前该把哪些"入口"交给接手人？
> **你**：本百科（总入口）+ 三个子百科（MCP / 声学后端 / 联网录音）+ 各业务 Reference skill + 各产品仓库 `README.md`。本文的两张路径表就是交接清单的骨架。

> **接手人**：我想让 AI 直接查检测数据 / 打标记，装什么？
> **你**：装本市场对应环境的变体（[about-smart-tpm-mcp](../../about-smart-tpm-mcp/SKILL.md)），文件下载再加姊妹插件；然后用业务 skill（quickstart 起步）。

### 作为离职交接文档时

本百科同时充当**离职交接总入口**。交接一个新接手人，建议按这个顺序移交阅读：

1. 本百科 systems/ 与 [04-key-concepts.md](./04-key-concepts.md) —— 建立全貌与黑话。
2. [09-tech-stack.md](./09-tech-stack.md) —— 知道每个系统用什么、在哪个仓库。
3. 本文（生态）—— 知道插件市场与各子百科的分工与入口。
4. 三个子百科 + 各业务 skill —— 按接手职责钻细分层。
5. 各产品仓库自身的 `README.md` / `CLAUDE.md` —— 代码级权威。

### 一句话记住这张地图

- **market（市场）** = 一个仓库，装配 MCP connector + 一排 skill，分 local/test/prod 三变体。
- **skill** 分两类：**百科型**（`about-*`，讲知识）与**业务型**（Reference 速查 / Task 干活）。
- **本百科** 站顶层给全貌与交接入口；**子百科** 各管一块细分层；被问细节就往子百科指路。
- **不写死的数字**：tool 数量以后端 `tools/list` 为准，skill 数量以仓库 `README.md` 为准。

> 生态会演进：新增变体、新增 skill、拆分子百科都可能发生。本文与仓库 `README.md` / `CHANGELOG.md` / marketplace 清单冲突时，**以后者为准**；skill 数量、tool 数量均不在本文写死。

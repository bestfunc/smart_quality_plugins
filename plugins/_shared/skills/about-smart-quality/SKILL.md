---
name: about-smart-quality
display_name: SmartQuality 智能质量检测平台 产品百科
description: SmartQuality（内部亦称 Smart TPM）产线声学质量检测平台的产品全貌与系统交接百科 — 给接手开发/新员工、做离职交接与验收的人、销售/CS、速查者查阅。覆盖交接涉及的全部系统：微信服务平台、数据同步(服务端+客户端)、安卓采集端、Web 检测平台、移动套装与 AI 插件层；含整体架构、领域概念、技术栈、生态子百科导航、交接对照表。能根据业务场景给出合理建议。
user-invocable: true
---

# SmartQuality 智能质量检测平台 产品百科

这是 **SmartQuality（内部亦称 Smart TPM）** 的**自助百科**。它把"产线上给产品录音 → 算法判 OK/NG → 沉淀数据集与标注 → 数据中心化 → AI 辅助查数据/追链路"这条完整产品链讲清楚。

**范围**：本百科聚焦"**产品全貌 + 离职交接涉及的各系统**"这一层——覆盖交接清单里的 6 个系统模块。各细分层（MCP 集成、声学后端、联网录音）另有专门的 **子百科**，本百科负责总览与导流，不重复它们的细节（见 [reference/12-ecosystem.md](./reference/12-ecosystem.md)）。

不写代码 / 不调用工具。要实际操作数据集/检测流程/算法/生产记录，请用本仓库的业务 skill；要下载文件走姊妹插件 SmartTPM_Files_Plugin。

---

## 我是谁？我从哪里看起？

### 🧑‍💻 我是接手开发 / 新员工，要快速理解这套系统

1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 一分钟看懂 SmartQuality 是什么、由哪些系统组成
2. **[reference/02-architecture.md](./reference/02-architecture.md)** — 端到端架构：一条录音从现场到 AI 要经过哪些系统
3. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — 检测流程/算法流/数据集/标记/工位分库 等黑话，后面所有文档都建立在这之上
4. **[scenarios/01-successor-onboarding.md](./scenarios/01-successor-onboarding.md)** — 接手第一周怎么走：搭环境 → 读哪个仓库 → 跑通一条链路
5. 你负责哪个系统就直接看 **[systems/](./systems/)** 下对应那篇

### 🔁 我要做 / 验收离职交接（对照协议清单核对）

1. **[faq/02-handover-crosswalk.md](./faq/02-handover-crosswalk.md)** — **交接对照表**：协议模块 ↔ 仓库 ↔ 本百科 ↔ 代码分支，逐项核对
2. **[systems/](./systems/)** 六篇 — 每篇末尾都有"接手人从哪读起代码"与"已知缺陷"，即交接文档正文
3. **[reference/09-tech-stack.md](./reference/09-tech-stack.md)** — 各仓库技术栈与版本，验收环境时对照

### 🎤 我是销售 / CS，要讲清"我们的平台 + AI 能力"

1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 一句话定位 + 价值主张
2. **[systems/06-mobile-suite.md](./systems/06-mobile-suite.md)** — 采集→分析→AI 的完整链路（最有演示冲击力的部分）
3. **[reference/12-ecosystem.md](./reference/12-ecosystem.md)** — AI 插件生态与子百科地图

### 📚 我只是想查个术语 / 缩写

直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference（事实层）— `reference/`
> 编号沿用通用百科模板（01-12）；未列出的 03/05/06/07/08/10/11 对应"版本对比/目标客户/竞品/商业模式/路线图/部署模式/AI集成独立章节"——本百科定位于内部产品交接、无对外销售/竞品诉求，故跳过。

| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 一句话定位、解决什么问题、产品由哪些系统组成的总表 |
| [02-architecture.md](./reference/02-architecture.md) | 端到端架构图，分层职责/技术栈/数据流/仓库归属 |
| [04-key-concepts.md](./reference/04-key-concepts.md) | 领域黑话：组织→项目→工位→检测流程→算法→数据集→标记→测试/生产记录 |
| [09-tech-stack.md](./reference/09-tech-stack.md) | 跨 8 个仓库的技术栈主表 + 逐系统说明 + 版本矩阵 |
| [12-ecosystem.md](./reference/12-ecosystem.md) | 插件 marketplace 结构、变体、全部 skill 清单、本百科与子百科分工 |

### Systems（交接系统层）— `systems/`
> 对应离职交接文档的 6 个系统模块，每篇=一份交接正文（含架构/配置/已知缺陷/从哪读起代码）。

| 文件 | 交接模块 | 主要仓库 |
|---|---|---|
| [01-wechat-service.md](./systems/01-wechat-service.md) | 微信服务平台 | wxdevops-task-admin, bestfunc_wechat_plugin |
| [02-data-sync-service.md](./systems/02-data-sync-service.md) | 数据同步服务 | sync-data（服务端） |
| [03-data-sync-client.md](./systems/03-data-sync-client.md) | 数据同步客户端 | sync-data（sync-client） |
| [04-app-client.md](./systems/04-app-client.md) | 安卓/桌面采集端 | smart_tpm_app（安卓）, SmartQuality-client（桌面） |
| [05-web-platform.md](./systems/05-web-platform.md) | Web 检测平台 | smart_tpm_web_v2 |
| [06-mobile-suite.md](./systems/06-mobile-suite.md) | 移动套装 + AI 分析链路 | Tinia_nodes_smart_quality_app, framework_v3_smart_quality, smart_quality_plugins |

### Scenarios（场景层）— `scenarios/`
| 文件 | 内容摘要 |
|---|---|
| [01-successor-onboarding.md](./scenarios/01-successor-onboarding.md) | 接手人上手工作流：搭环境→读代码顺序→跑通一条链路 |
| [02-end-to-end-dataflow.md](./scenarios/02-end-to-end-dataflow.md) | 一条录音端到端：采集→同步→检测→标注→AI 查询 |

### FAQ（话术层）— `faq/`
| 文件 | 内容摘要 |
|---|---|
| [01-successor-questions.md](./faq/01-successor-questions.md) | 接手开发常见问题 + 已知文档待校对项 |
| [02-handover-crosswalk.md](./faq/02-handover-crosswalk.md) | 交接对照表：协议模块↔仓库↔本百科↔代码分支 |

### Glossary（术语层）— `glossary/`
| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全域术语与缩写，按 6 个域分组 |

---

## 当你被问到的时候

> **新同事**：SmartQuality 和 Smart TPM 是两个产品吗？
> **你**：同一产品套装的两个内部叫法。看 [glossary](./glossary/terms.md) 第一条 + [reference/01](./reference/01-product-overview.md)。

> **接手人**：安卓端代码在哪个仓库哪个分支？
> **你**：`smart_tpm_app`，实际主线在 `feature/app-v2`（master 停在一年前）；`SmartQuality-client` 是 Electron 桌面端，不打安卓包。详见 [systems/04](./systems/04-app-client.md) 与 [faq/02](./faq/02-handover-crosswalk.md)。

> **交接验收人**：怎么核对协议第四条列的东西都交清了？
> **你**：用 [faq/02-handover-crosswalk.md](./faq/02-handover-crosswalk.md) 的对照表逐项对，每个系统再翻 [systems/](./systems/) 对应篇的"已知缺陷/从哪读起"。

> **开发**：一条产线录音最后怎么进到数据集给算法用？
> **你**：看 [scenarios/02-end-to-end-dataflow.md](./scenarios/02-end-to-end-dataflow.md)，一图串起采集→同步→检测→标注。

> **任何人**：某个黑话（如 detect_flow / 工位分库 / block / sqMode）是什么？
> **你**：直接查 [glossary/terms.md](./glossary/terms.md)。

---

## 维护说明（给写 skill 的人看）

- 事实来源：交接涉及的各仓库代码与 README —— smart_tpm_web_v2 / SmartQuality-api / SmartQuality-client / smart_tpm_app / sync-data / wxdevops-task-admin / bestfunc_wechat_plugin / Tinia_nodes_smart_quality_app / framework_v3_smart_quality / smart_quality_plugins / bi。
- 本百科讲**全貌与交接**；MCP 集成层、声学后端、联网录音等细分层以 **子百科**（`../about-smart-tpm-mcp/`、`../about-audio-quality/`、`../about-net-audio/`）为准，本百科只导流不复制。
- 凡涉及"未来"用 *计划 / 设想 / 在调研* 修饰，不用 *承诺 / 一定 / 必须 / 即将上线*。
- 跨文档引用一律相对路径，方便 GitHub 渲染。
- 严禁写入：真实客户名/现场名、内网 IP/凭据/密钥、营收/成本、团队成员姓名/薪酬。
- 新增概念时同步更新 [glossary/terms.md](./glossary/terms.md)。
- 部分系统文档标注了"文档漂移/待校对"（如 sync-data README 与实现不一致、SmartQuality-client 技术栈），修订时以代码为唯一事实来源。

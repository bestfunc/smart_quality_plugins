---
name: about-smartplc
display_name: SmartPLC 工业声学质检系统 产品百科
description: SmartPLC(代码库 BestPLC)的方方面面 — 给开发/新员工、销售/客户成功、现场实施运维、做评估的潜在客户查阅。涵盖产品定位、系统架构、PLC 多协议适配、检测工作流、对外集成、技术栈、部署配置、目标客户、版本交付、演进路线。能根据业务场景给出使用建议。
user-invocable: true
---

# SmartPLC 工业声学质检系统 产品百科

这是 **SmartPLC**(代码库 `BestPLC`,出品方"巅峰表现")的**自助百科**。无论你是来上手代码的开发、要话术的销售、去现场布点的实施运维,还是在评估这套系统的潜在客户,都能从这里查到产品知识,并根据业务场景拿到合理建议。

一句话定位:**装在产线工控机上的 WPF/.NET 8 桌面应用,做工业产线多工位声学质检自动化——用 S7-1500 / Modbus TCP / 欧姆龙 FINS TCP 三种协议实时读写 PLC,驱动录音盒子采音、送 AI 算法判 OK/NG,再把结果写回 PLC 控制产品流转。**

> 本 skill **不写代码 / 不调用工具**。需要改代码请用项目里的开发类 skill(如 `smartplc-perf` 性能诊断)。

---

## 我是谁?我从哪里看起?

### 🧑‍💻 我是开发同事 / 新员工,要快速读懂这套系统
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 先搞清这是干嘛的、解决什么问题
2. **[reference/02-architecture.md](./reference/02-architecture.md)** — 分层架构 + 核心模块职责,建立全局地图
3. **[reference/05-detection-workflow.md](./reference/05-detection-workflow.md)** — 主/子流程 8 阶段状态机,这是系统的心脏
4. **[reference/03-key-concepts.md](./reference/03-key-concepts.md)** — 工位/检测点/通道/阶段等概念,读代码前先扫一遍
5. **[reference/07-tech-stack.md](./reference/07-tech-stack.md)** — 技术栈与关键依赖
6. **[faq/02-developer-questions.md](./faq/02-developer-questions.md)** — 常见困惑(为什么数据库直连?多工位并发怎么扛?)

### 🧑‍💼 我是销售 / 客户成功(或在评估这套系统的潜在客户)
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 价值主张、解决的痛点
2. **[reference/09-target-customers.md](./reference/09-target-customers.md)** — 适用行业/角色画像,判断是否匹配
3. **[reference/04-plc-protocols.md](./reference/04-plc-protocols.md)** — 支持哪些 PLC(客户最常问的兼容性)
4. **[reference/10-editions-delivery.md](./reference/10-editions-delivery.md)** — 版本线、客户专版、交付与发版节奏
5. **[faq/01-customer-questions.md](./faq/01-customer-questions.md)** — 客户常问问题的标准回答
6. **[faq/03-sales-talking-points.md](./faq/03-sales-talking-points.md)** — 卖点话术 *(内部参考,不对外发)*

### 🔧 我是现场实施 / 运维,要把它在产线跑起来
1. **[reference/08-deployment-config.md](./reference/08-deployment-config.md)** — 部署形态 + 全部配置项怎么填
2. **[reference/04-plc-protocols.md](./reference/04-plc-protocols.md)** — 选协议、配点位、连接参数
3. **[reference/06-integrations.md](./reference/06-integrations.md)** — 对接录音盒子 / 算法引擎 / MySQL / MinIO 的地址与开关
4. **[scenarios/01-line-operator-setup.md](./scenarios/01-line-operator-setup.md)** — 上线一条产线的完整步骤
5. **[scenarios/03-field-implementation.md](./scenarios/03-field-implementation.md)** — 现场对接客户 PLC 的实战流程

### 📚 我只是想查个术语 / 概念
直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference(事实层)— `reference/`
| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 产品定位、解决的痛点、核心价值、用户角色 |
| [02-architecture.md](./reference/02-architecture.md) | 分层架构(UI / 工作流引擎 / 协议适配 / 集成 / 横切)、核心模块职责、关键数据流 |
| [03-key-concepts.md](./reference/03-key-concepts.md) | 核心概念:检测工位 / 检测点 / 智能设备 / 通道 / 阶段 / 主子流程 / 灵敏度 / 动态点位 |
| [04-plc-protocols.md](./reference/04-plc-protocols.md) | 三协议适配(S7-1500 / Modbus TCP / 欧姆龙 FINS TCP)、点位模型、连接池与重连 |
| [05-detection-workflow.md](./reference/05-detection-workflow.md) | 检测主流程 / 子流程、8 阶段状态机、信号驱动时序、结果写回 |
| [06-integrations.md](./reference/06-integrations.md) | 对外集成:SmartAudio 录音盒子 / 算法引擎 / SmartQuality(MySQL 直连)/ MinIO |
| [07-tech-stack.md](./reference/07-tech-stack.md) | 运行时(.NET 8 WPF)、UI 框架、关键 NuGet 依赖及用途、工业通信库 |
| [08-deployment-config.md](./reference/08-deployment-config.md) | 部署形态(单机现场)、Settings 配置项分类详解、外部服务连接清单 |
| [09-target-customers.md](./reference/09-target-customers.md) | 目标客户画像(适用行业 / 产线规模 / 使用角色),不含真实客户名 |
| [10-editions-delivery.md](./reference/10-editions-delivery.md) | 版本线(v3.2/3.3/3.5)、客户专版机制、release-YYMMDD 发版节奏;商业报价 `<TODO 待补充>` |
| [11-roadmap.md](./reference/11-roadmap.md) | 演进路线(v4 工位微服务化 + 配置数据驱动 + Web 可观测,弱措辞) |
| [12-competitive-landscape.md](./reference/12-competitive-landscape.md) | 竞品格局**参考稿**(联网核实受限,待坐实):声学 vs 视觉赛道区分、NVH 测量厂商 vs AI 视觉平台、SmartPLC 定位差异 |

### Scenarios(场景层)— `scenarios/`
| 文件 | 内容摘要 |
|---|---|
| [01-line-operator-setup.md](./scenarios/01-line-operator-setup.md) | 产线运维:配工位 / PLC 点位 / 设备地址 / 灵敏度,上线一条产线 |
| [02-quality-engineer-review.md](./scenarios/02-quality-engineer-review.md) | 质量工程师:查检测结果、分析 OK/NG 分布、调灵敏度 |
| [03-field-implementation.md](./scenarios/03-field-implementation.md) | 现场实施:选 PLC 协议、按点位表配点、联调录音盒子 + 算法引擎 |

### FAQ(话术层)— `faq/`
| 文件 | 内容摘要 |
|---|---|
| [01-customer-questions.md](./faq/01-customer-questions.md) | 对外:客户 / 评估者常问(支持哪些 PLC?怎么部署?数据放哪?) |
| [02-developer-questions.md](./faq/02-developer-questions.md) | 对内开发:数据库直连原因、多工位并发、算法接口调用 |
| [03-sales-talking-points.md](./faq/03-sales-talking-points.md) | 对内销售卖点话术 *(内部参考,不对外发)* |

### Glossary(术语层)— `glossary/`
| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全部专有名词 + 缩写表(检测工位、检测点、智能设备、霍尔信号、dataContent 等) |

---

## 当你被问到的时候

> **潜在客户**:你们支持我现场的欧姆龙 PLC 吗?
> **你**:看 `reference/04-plc-protocols.md`,SmartPLC 通过统一适配层支持 Siemens S7-1500、Modbus TCP、欧姆龙 FINS TCP 三种协议,可直接引用兼容矩阵。

> **新来的开发**:一个产品从上料到出结果,代码里是怎么走的?
> **你**:看 `reference/05-detection-workflow.md`,里面有主流程(StableWorkflow)→ 子流程(StableWorkChildflow)的 8 阶段时序,以及每阶段"读 PLC → 录音 → 算法 → 写回"的动作链。

> **现场实施**:这台机器要连哪些外部服务、配置在哪填?
> **你**:看 `reference/08-deployment-config.md`,Settings 配置项按 PLC/检测/数据库/MinIO/算法分类列全了;对外地址看 `reference/06-integrations.md`。

> **销售**:这套系统比人工质检强在哪?
> **你**:看 `reference/01-product-overview.md` 的核心价值 + `faq/03-sales-talking-points.md` 的卖点话术(内部)。

---

## 维护说明(给写 / 改 skill 的人看)

- **事实来源**:本仓库代码(`BestPLC/`)+ `BestPLC/Docs/`、`docs/`、`doc/` 三层文档 + git 历史。凡涉及"未来路线"用*计划 / 路线图 / 设想*修饰,不用*承诺 / 一定 / 必须 / 即将上线*。
- **裁剪红线**:不写真实客户名(代码分支 / tag / 版本后缀里出现的现场标识一律脱敏)、不写营收 / 团队规模 / 融资、不写数据库凭据 / 内部 IP / MinIO 密钥、不写内部决策辩论。`faq/03-sales-talking-points.md` 标题已标"不对外发"。
- **跨文档引用**一律相对路径,方便 GitHub 渲染。
- **内容更新**时同步更新 `glossary/terms.md`(如有新概念)。
- **竞品章节**为参考稿(`reference/12-competitive-landscape.md`):因联网核实受限,内容多为公开常识/训练知识,已逐条标注待核;联网恢复后应逐家上官网坐实,尤其中国本土声学异响产线方案商的点名。
- **写作状态**:本文件为入口蓝图;reference / scenarios / faq / glossary 各文件待分批撰写后逐步落地(届时本索引应无死链)。

---
name: about-audio-quality
display_name: AI 音频质检系统 产品百科
description: AI 音频质检系统的方方面面 — 给开发/新员工、运维/实施、速查术语的同事查阅。涵盖 DAG 流程引擎、检测引擎与双视角聚合、21+ 算法模型、三级参数解析、4 GPU 集群与发布运维、技术栈、路线图。能根据业务/排障需求给出定位建议。
user-invocable: true
---

# AI 音频质检系统 产品百科

这是 **AI 音频质检系统**（代码仓 `ai_api_server_v2`）的**对内自助百科**。无论你是刚接手的开发、负责发布的运维，还是只想查个术语的同事，都能从这里查到产品知识与上下文，并据此定位到对应代码/文档。

**这是知识检索结构，不是开发工具**：本 skill 不写代码、不调用工具。要写代码/排障，去用项目里实际存在的开发类 skill（`add-endpoint` / `debug-assistant` / `hotfix` / `code-review` / `perf-analysis` / `db-optimize` 等）。

> 适用范围：**纯对内**。正文含内网地址/端口/服务器/已知踩坑（与 `docs/`、`CLAUDE.md` 一致），但**不含任何真实凭据**（SSH key / API Key / DB 密码一律环境变量占位）。

---

## 我是谁？我从哪里看起？

### 🧑‍💻 我是开发 / 新员工，想搞懂这套系统怎么跑起来

按这个顺序看：
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 先建立"这是什么、能干什么、边界在哪"的整体认知
2. **[reference/02-architecture.md](./reference/02-architecture.md)** — 看请求从 nginx 进来到结果返回的完整链路，定位每个模块
3. **[reference/03-key-concepts.md](./reference/03-key-concepts.md)** — DAG 引擎 / 9 类节点 / 三级参数 / mode 动态分发，这些是改代码绕不开的核心机制
4. **[scenarios/01-developer-onboarding.md](./scenarios/01-developer-onboarding.md)** — 本地跑起来 + 跑测试 + 怎么加节点 / 加算法的实操路径
5. **[reference/05-algorithm-models.md](./reference/05-algorithm-models.md)** — 21+ 种 BF_* 算法各自干什么、需不需要 GPU

### 🛠️ 我是运维 / 实施，负责部署、发布和排障

按这个顺序看：
1. **[reference/06-deployment-cluster.md](./reference/06-deployment-cluster.md)** — 4 GPU 集群结构、nginx 分流、发布三件套（先看这个，避免误碰老容器）
2. **[scenarios/02-ops-deploy-troubleshoot.md](./scenarios/02-ops-deploy-troubleshoot.md)** — 标准发布流程 + 排障第一步（超时 / hang / 算法异常被吞）
3. **[reference/04-detect-engine.md](./reference/04-detect-engine.md)** — `/detect/execute` 协议、双视角聚合、warnings 怎么读
4. **[faq/02-ops-questions.md](./faq/02-ops-questions.md)** — 高频运维问题速答（reload 超时、worker 不一致、env 不生效）
5. **[reference/07-tech-stack.md](./reference/07-tech-stack.md)** — 依赖版本与外部系统（MySQL / MinIO / Qdrant）连接要求

### 📚 我只是想查个术语 / 概念

直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference（事实层）— `reference/`
| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 一句话定位、核心能力清单、已支持 / 不支持功能、性能限制 |
| [02-architecture.md](./reference/02-architecture.md) | 整体架构拓扑、请求处理链路、各模块职责、完整数据流 |
| [03-key-concepts.md](./reference/03-key-concepts.md) | DAG 引擎、9 类节点契约、三级参数解析、mode 动态分发、热加载 |
| [04-detect-engine.md](./reference/04-detect-engine.md) | 检测引擎、dataContent 协议、双视角聚合（point_first / stage_first）、warnings |
| [05-algorithm-models.md](./reference/05-algorithm-models.md) | 21+ 种 BF_* 算法清单、模型制品（pt + yaml 配套）、MinIO / Qdrant 用途 |
| [06-deployment-cluster.md](./reference/06-deployment-cluster.md) | 部署形态（单机 / 集群 / Jetson）、4 GPU 集群、nginx 分流、发布运维 |
| [07-tech-stack.md](./reference/07-tech-stack.md) | 语言 / 框架 / 推理栈 / 数据层 / 外部系统版本与连接要求 |
| [08-roadmap.md](./reference/08-roadmap.md) | 计划中 / 在调研的能力（MCP 网关、异步队列、多语言 SDK 等，弱措辞） |

### Scenarios（场景层）— `scenarios/`
| 文件 | 内容摘要 |
|---|---|
| [01-developer-onboarding.md](./scenarios/01-developer-onboarding.md) | 新人从 0 跑起来、跑测试、加 DAG 节点 / 加算法模型的实操路径 |
| [02-ops-deploy-troubleshoot.md](./scenarios/02-ops-deploy-troubleshoot.md) | 标准发布流程、滚动重启、四类典型故障的排障第一步 |
| [03-add-algorithm-flow.md](./scenarios/03-add-algorithm-flow.md) | 算法迭代：模型制品上传 → 关联流程 → reloadOne 热加载 → 验证 |

### FAQ（话术层）— `faq/`
| 文件 | 内容摘要 |
|---|---|
| [01-developer-questions.md](./faq/01-developer-questions.md) | 开发高频问题：节点契约、参数优先级、mode 透传、子流程嵌套 |
| [02-ops-questions.md](./faq/02-ops-questions.md) | 运维高频问题：reload 超时、worker 不一致、env 不生效、GPU 占用 |

### Glossary（术语层）— `glossary/`
| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全部专有名词 + 缩写表（DAG / mode / BF_* / point_first / stageAssignments 等） |

---

## 当你被问到的时候

> **新同事**：这个流程图里的 `classify` 节点到底干嘛？
> **你**：先看 [reference/03-key-concepts.md](./reference/03-key-concepts.md) 的节点契约表，`classify` 是 one→one 的阈值判定，输出 `ok`/`ng` 驱动条件边路由；再看 [glossary/terms.md](./glossary/terms.md) 查 `_condition` 字段含义。

> **运维**：生产请求大面积超时，先查什么？
> **你**：照 [scenarios/02-ops-deploy-troubleshoot.md](./scenarios/02-ops-deploy-troubleshoot.md) 的排障第一步：先 `curl .../cluster/workers` 看 4/4 healthy，再 `nvidia-smi` 看 4 卡，再 grep ConcurrencyMonitor 看数字是否在变（不变=hang）。

> **对接方对接前问**：`/detect/execute` 要传哪些数据？
> **你**：看 [reference/04-detect-engine.md](./reference/04-detect-engine.md) 的 dataContent 五张表协议 + `flowNumber`/`versionCode`/`mode` 字段说明，响应结构看双视角聚合段。

> **开发**：我改了算法参数，为什么没生效？
> **你**：看 [reference/03-key-concepts.md](./reference/03-key-concepts.md) 三级参数解析——前端 > 数据库 > 本地 JSON，高优先级覆盖低优先级；改完要发布流程 + reloadOne 才生效，详见 [faq/01-developer-questions.md](./faq/01-developer-questions.md)。

> **新员工**：这套系统都能检测什么？
> **你**：看 [reference/05-algorithm-models.md](./reference/05-algorithm-models.md)，21+ 种算法覆盖音频分类、多维评分、特征提取、异常检测、脉冲/电机检测、分贝测量、感知评估、多算法仲裁。

---

## 维护说明（给写 / 改 skill 的人看）

- **事实来源**：代码仓 `v2/`（dag / detect / flow / models / api）+ `docs/`（product_architecture_doc / detect_execute_api / prod_deploy_guide 等）+ `README.md` + `CLAUDE.md` + git log / tag。不发明数字、不虚构客户、不臆测竞品。
- **凭据隔离**：正文出现连接信息时用环境变量名占位（如 `${MINIO_ENDPOINT}`），绝不写真实 key / 密码。
- **未来路线弱措辞**：凡"未来能力"用 *计划 / 路线图 / 设想 / 在调研*，不用 *承诺 / 一定 / 必须 / 即将上线*。
- **跨文档引用一律相对路径**，方便 GitHub 渲染。
- **内容更新时同步 glossary/terms.md**（出现新概念就补术语）。
- **版本/架构变化时优先核对**：`docs/prod_deploy_guide.md`、`CLAUDE.md` 红线表 —— 它们是生产配置的权威来源，本百科与之冲突时以它们为准。

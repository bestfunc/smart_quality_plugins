---
name: about-smart-tpm-mcp
display_name: Smart TPM MCP 集成百科
description: Smart TPM 检测平台 Claude Code 插件 / MCP 集成层的方方面面 — 给插件用户、仓库维护者、销售演示者、速查者查阅。涵盖 marketplace 结构、3 个变体（local/test/prod）、MCP HTTP transport、OAuth 2.1 + PKCE + DCR 鉴权、scope 模型、工具一览（以后端 tools/list 为准）、9 个业务 skill 导览、多客户端兼容（Claude Code / Codex / Qwen CLI）。能根据使用场景给出合理建议。
user-invocable: true
---

# Smart TPM MCP 集成百科

这是 **Smart TPM MCP 集成层** 的自助百科。不管你是来本地接入插件的工程师、维护这个 marketplace 仓库的开发、给客户演示 AI 能力的销售，还是只想查个术语的同事，都能从这里找到你要的东西，并据此给出合理建议。

**范围限定**：本百科聚焦"插件 + MCP 集成"这一层（本仓库 `SmartTPM_Plugins/` 的全部产物）。Smart TPM 检测平台本身的产品全貌、前后端实现、业务流程，请查阅平台仓库各自的文档与 skill。

不写代码 / 不调用 MCP 工具。如果你要实际操作数据集 / 检测流程 / 测试任务 / 算法，请用本仓库的 9 个业务 skill（1 Onboarding + 3 Reference + 5 Task，详见下方索引）。

---

## 我是谁？我从哪里看起？

### 🧑‍💻 我是本地工程师，刚被推荐用这个插件

按这个顺序看：
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 一分钟知道插件是什么、为什么用它、能干哪些事
2. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — 搞清 MCP / Scope / Skill / Tool 4 个核心概念，后面所有文档都建立在这之上
3. **[scenarios/01-engineer-onboarding-workflow.md](./scenarios/01-engineer-onboarding-workflow.md)** — 端到端走一遍：装插件 → OAuth 授权 → 用第一个 skill 查数据集
4. **[reference/11-ai-integration.md](./reference/11-ai-integration.md)** — 你用的客户端（Claude Code / Codex / Qwen CLI）有什么差异和坑
5. 遇到具体问题 → **[faq/01-user-questions.md](./faq/01-user-questions.md)**

### 🛠️ 我是仓库维护者，要发版 / 改配置 / 加 skill

按这个顺序看：
1. **[reference/02-architecture.md](./reference/02-architecture.md)** — marketplace + _shared + 3 变体 的物理布局与同步规则
2. **[reference/03-edition-comparison.md](./reference/03-edition-comparison.md)** — local / test / prod 三变体差异（URL、scope、谁用）
3. **[scenarios/02-maintainer-release-workflow.md](./scenarios/02-maintainer-release-workflow.md)** — 发版流程：改 `plugin.json` → bump version → _shared 同步到 3 变体 → push → tag
4. **[reference/09-tech-stack.md](./reference/09-tech-stack.md)** — 仓库技术栈与依赖（其实只有 markdown + JSON，但有几条约束）
5. **[reference/08-roadmap.md](./reference/08-roadmap.md)** — 在调研 / 计划中的能力
6. 维护中遇到具体坑 → **[faq/02-developer-questions.md](./faq/02-developer-questions.md)**

### 🎤 我是销售 / CS，要给客户演示"我们的平台能让 AI 直接操作"

按这个顺序看：
1. **[reference/01-product-overview.md](./reference/01-product-overview.md)** — 一句话定位 + 价值主张
2. **[scenarios/03-sales-demo-workflow.md](./scenarios/03-sales-demo-workflow.md)** — 现场演示推荐顺序（哪个 skill 最有视觉冲击力、避开哪些坑）
3. **[reference/11-ai-integration.md](./reference/11-ai-integration.md)** — 兼容客户端清单（客户问"我们已经用 X 客户端能不能接"）
4. **[faq/03-sales-talking-points.md](./faq/03-sales-talking-points.md)** *(内部参考，不对外发)*

### 📚 我只是想查个术语 / 缩写

直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference（事实层）— `reference/`

> 编号沿用通用百科模板（01-12），未列出的 05 / 06 / 07 / 10 / 12 对应"目标客户 / 竞品对比 / 商业模式 / 部署模式独立章节 / 生态"等章节，本项目（内部插件、无 B2B 销售、单一部署形态、本身就是插件）不适用，故跳过。

| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 插件是什么 / 解决什么问题 / 使用对象 / 一句话价值主张 |
| [02-architecture.md](./reference/02-architecture.md) | marketplace + `_shared/skills` + 3 变体物理布局；MCP HTTP transport 数据流 |
| [03-edition-comparison.md](./reference/03-edition-comparison.md) | local / test / prod 三变体对比表（URL、用途、谁该装哪个） |
| [04-key-concepts.md](./reference/04-key-concepts.md) | MCP / Scope / Skill / Tool / OAuth 2.1 + PKCE + DCR — 5 个核心概念解释 |
| [08-roadmap.md](./reference/08-roadmap.md) | 在调研 / 计划中的能力（detect_flows:write、更多 skill、生态…） |
| [09-tech-stack.md](./reference/09-tech-stack.md) | 仓库技术构成（markdown + JSON）、Claude Code plugin spec、约束清单 |
| [11-ai-integration.md](./reference/11-ai-integration.md) | Claude Code / Codex / Qwen CLI 多客户端兼容性、`httpUrl` 兼容字段的来源 |

### Scenarios（场景层）— `scenarios/`

| 文件 | 内容摘要 |
|---|---|
| [01-engineer-onboarding-workflow.md](./scenarios/01-engineer-onboarding-workflow.md) | 工程师本地接入：装插件 → OAuth 授权 → 用 quickstart skill 跑通第一个查询 |
| [02-maintainer-release-workflow.md](./scenarios/02-maintainer-release-workflow.md) | 维护者发版：改 `_shared/skills` → 同步到 3 变体 → bump version → commit → push |
| [03-sales-demo-workflow.md](./scenarios/03-sales-demo-workflow.md) | 销售演示：推荐流程、视觉冲击点、典型客户问题应对 |

### FAQ（话术层）— `faq/`

| 文件 | 内容摘要 |
|---|---|
| [01-user-questions.md](./faq/01-user-questions.md) | 本地工程师常问：装不上 / 授权失败 / tool 不可见 / scope 不够 |
| [02-developer-questions.md](./faq/02-developer-questions.md) | 维护者常问：_shared 与变体同步策略、为什么不用 symlink、版本号怎么 bump |
| [03-sales-talking-points.md](./faq/03-sales-talking-points.md) | *(内部参考，不对外发)* 销售话术、价值主张、客户常见疑虑应答 |

### Glossary（术语层）— `glossary/`

| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全部专有名词 + 缩写表（MCP / PKCE / DCR / Scope / Skill / Tool / Transport / Marketplace 等） |

---

## 当你被问到的时候

> **本地工程师**：装完插件，Claude Code 没显示 smart-tpm 的 tool，怎么办？
> **你**：先看 [faq/01-user-questions.md](./faq/01-user-questions.md) 的"tool 不可见"一节，里面列了 4 个常见原因（未授权 / scope 不够 / 客户端缓存 / 错装了 local 变体）。如果都不命中，让用户提供 `claude mcp list` 输出。

> **客户演示中**：客户问"这跟你们 Web 界面比强在哪？"
> **你**：去 [faq/03-sales-talking-points.md](./faq/03-sales-talking-points.md) 的"AI 编排 vs 手动点击"段，里面有 3 个核心论点 + 一个"批量复现客户报错"的具体例子。

> **维护者**：要加一个新 skill，文件应该放哪、要不要复制到 3 个变体？
> **你**：看 [scenarios/02-maintainer-release-workflow.md](./scenarios/02-maintainer-release-workflow.md) 第 2 步"_shared 同步规则"，原件放 `_shared/skills/<新名>/`，然后物理复制到 3 个变体目录（不用 symlink，Windows 上不稳）。

> **新人**：MCP / Scope / Tool / Skill 有啥区别？
> **你**：直接给 [reference/04-key-concepts.md](./reference/04-key-concepts.md)，看完 5 分钟就分清。

> **销售**：客户说他们已经在用 Qwen Code 了，能不能接？
> **你**：能。看 [reference/11-ai-integration.md](./reference/11-ai-integration.md) 的"Qwen Code 兼容"段，1.0.2 版起 `plugin.json` 加了 `httpUrl` 字段就是为了它（Qwen 对 SSE 误识别问题）。

---

## 维护说明（给写 skill 的人看）

- **内容跨多文件，事实来源**：本仓库 `README.md` / `CHANGELOG.md` / `plugins/<变体>/.claude-plugin/plugin.json` / `plugins/_shared/skills/<各 skill>/SKILL.md` / `git log`。Smart TPM 平台业务概念以本仓库 `smart-tpm-business-concepts` skill 为权威。
- **未来路线用弱措辞**：用 "计划 / 路线图 / 设想 / 在调研"，**不用** "承诺 / 一定 / 必须 / 即将上线"。
- **跨文档引用一律相对路径**：`[xxx](./reference/02-architecture.md)` 而非绝对 URL。
- **敏感信息隔离**：`faq/03-sales-talking-points.md` 标题写 `*(内部参考，不对外发)*`；其他文件对外可读。
- **不出现**：团队成员姓名 / git author / 邮箱 / 客户真实名 / 融资 / 营收 / 团队规模 / 内部失败决策辩论。
- **保留**：内网 IP（192.168.2.121 / 192.168.2.175）与生产域名（smartquality.bestfunc.com）—— 用户已确认可写入。
- **同步术语表**：reference / scenarios / faq 中每出现一个**首次新概念**，要在 `glossary/terms.md` 补一行。
- **变体分发**：本 skill 原件位于 `plugins/_shared/skills/about-smart-tpm-mcp/`。如需让某变体的插件用户能直接 `/skill about-smart-tpm-mcp` 调用，物理复制一份到 `plugins/smart-tpm-{local,test,prod}/skills/about-smart-tpm-mcp/`（沿用现有 9 个 skill 的同步做法，不用 git symlink）。

# 01 — 产品总览

## 定义

**Smart TPM Plugins** 是给 Smart TPM 检测平台配套的 Claude Code 插件市场（marketplace），通过 MCP（Model Context Protocol）把平台后端的能力暴露给本地 AI 客户端，让工程师 / 数据 / 算法同事用**自然语言**驱动 AI 完成原本要跨多个页面才能做的"查、对照、初稿配置"。

仓库本体是一个 Claude Code marketplace：根目录 `.claude-plugin/marketplace.json` 声明三个变体（`smart-tpm-local` / `smart-tpm-test` / `smart-tpm-prod`），每个变体各自挂一个 MCP server，并附带 9 个共享的 skill。

## 解决什么问题

Smart TPM Web 提供完整的页面化操作能力，但工程师在以下场景下被频繁迫使**手动跨页跳转**：

| 痛点 | 手动做法 | 用插件 |
|---|---|---|
| 想复盘一条客户报错 | 打开测试记录 → 跳到检测流程 → 翻执行记录 → 找算法版本 | 一句话："给我复现 channel X 这条记录"，AI 自动串起 4 个查询 |
| 想仿一份测试任务 | 打开旧任务 → 抄参数 → 新建 → 一项项填 | "仿 #1024 模板新建一份，sampleRatio 改成 0.3" |
| 想抽样看标注 | 数据集页 → 按标签筛 → 翻页 → 点开标注 | "抽数据集 X 里 label=NG 的前 20 条" |
| 想搞清楚某节点用了什么算法 | 检测流程页 → 节点详情 → 算法流页 → 算法版本 | "这个节点跑的什么算法链路？" |

**插件不是替代 Web，而是给"读多 + 编排多"的场景加一层 AI 编排。** 真正的写操作 95% 仍在 Web 完成；插件目前只开放一个写入口（基于模板新建测试任务草稿）。

## 使用对象

| 角色 | 典型来意 |
|---|---|
| **本地工程师**（数据 / 算法 / 测试） | 安装其中一个变体，OAuth 授权后用 skill 完成日常查询 / 复盘 / 仿任务 |
| **仓库维护者** | 维护 marketplace 结构，发版，加 skill，同步 `_shared` 到 3 个变体 |
| **销售 / CS 演示者** | 客户面前演示"我们的平台支持 AI 直接驱动"——快速可视化 5 个模块的串联编排 |
| **速查者** | 临时查个术语 / 缩写 / 工具名 / scope 含义 |

## 一句话价值主张

> Smart TPM Plugins = **把 Smart TPM 后端 4 个核心模块（数据集 / 检测流程 / 测试任务 / 算法）+ 1 个生产数据模块（detect_records）的约 20 个查询能力，通过 MCP + OAuth 暴露给本地 AI，配 9 个 skill 把高频跨页操作压成一句自然语言。**

## 关键事实

- **9 个 skill** = 1 Onboarding + 3 Reference + 5 Task（详见 [02-architecture.md](./02-architecture.md)）
- **5 个模块** = datasets / detect_flows / test_tasks / algorithms / detect_records
- **MCP tool**：分布在 5 个模块,具体名单以后端 `tools/list` 为准
- **3 个变体** = local（本机 docker）/ test（内网 121）/ prod（内网 175）
- **写操作只有 1 个**：`test_tasks_create_from_template` —— 其余全是 `:read`
- **鉴权**：OAuth 2.1 + PKCE + DCR，浏览器一次同意即可，无需复制粘贴 token
- **客户端兼容**：Claude Code / Codex / Qwen Code CLI（详见 [11-ai-integration.md](./11-ai-integration.md)）

## 不在范围

本插件 / 本百科**不覆盖**：

- Smart TPM 检测平台本身的产品功能（前后端、业务流程、UI 设计）—— 那是平台仓库的事
- 算法本身的实现 / 训练 / 评估方法
- 数据集的标注规范与质量控制流程
- 工位部署 / 硬件运维 / 现场施工

需要这些信息时，回到 Smart TPM 平台仓库的对应文档与 skill（如 `smart-tpm-business-concepts`）。

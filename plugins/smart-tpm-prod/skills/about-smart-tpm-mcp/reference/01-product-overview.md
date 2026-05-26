# 01 — 产品总览

## 定义

**Smart TPM Plugins** 是 Smart TPM 检测平台配套的 Claude Code 插件市场（marketplace），通过 MCP（Model Context Protocol）把后端 9 个模块、61 个业务 tool 暴露给工程师本地 AI 客户端，让数据 / 算法 / 测试 / CS 同事用**自然语言**驱动 AI 完成原本要跨多个 Web 页面才能做的"查、对照、复盘、起草、打标"。

仓库本体是一个 Claude Code marketplace：根目录 `.claude-plugin/marketplace.json` 声明三个变体（`smart-tpm-local` / `smart-tpm-test` / `smart-tpm-prod`），每个变体各自挂一个 MCP server，并附带 11 个共享 skill（10 业务 + 本百科）。

文件下载场景（MinIO bucket / 数据集音频 / 模型 artifact）走姊妹插件 [SmartTPM_Files_Plugin](https://github.com/bestfunc/SmartTPM_Files_Plugin)，两个 plugin 指向同一个 MCP server，**OAuth 一次浏览器同意两个都生效**。

## 解决什么问题

Smart TPM Web 提供完整的页面化操作能力，但工程师在以下场景下被频繁迫使**手动跨页跳转 / 手动重复点按**：

| 痛点 | 手动做法 | 用插件 |
|---|---|---|
| 复盘客户报错 | 打开测试记录 → 跳到检测流程 → 翻执行记录 → 找算法版本 | 一句话："给我复现 channel X 这条记录"，AI 自动串 `detect_records_trace` 一键拿 3 层链路 |
| 仿一份测试任务 | 打开旧任务 → 抄参数 → 新建 → 一项项填 | "仿 #1024 模板新建一份，sampleRatio 改成 0.3" |
| 抽样看标注 | 数据集页 → 按标签筛 → 翻页 → 点开标注 | "抽数据集 X 里 label=NG 的前 20 条" |
| 弄清某节点用什么算法 | 检测流程页 → 节点详情 → 算法流页 → 算法版本 | "这个节点跑的什么算法链路？" |
| 给样本批量打粗标 / 细标 | dataset annotate 页 → 右键 → 选 markCode → 一条条点 | "把这 200 个 channel 全标成 OK"（`marks_batch_set_rough`） |
| 看 AI 引擎调用快照 | 不便查 / 要去 ELK | `audio_ai_req_logs_list` 单 tool 取回 |

**插件不是替代 Web，而是给"读多 + 编排多 + 重复点击多"的场景加一层 AI 编排。** 真正的复杂表单操作 90% 仍在 Web 完成；插件目前开放 6 个写入口（1 个新建任务草稿 + 5 个样本打标），全部走 service 层 + 落审计表。

## 使用对象

| 角色 | 典型来意 |
|---|---|
| **本地工程师**（数据 / 算法 / 测试） | 安装其中一个变体，OAuth 授权后用 skill 完成日常查询 / 复盘 / 仿任务 / 打标 |
| **仓库维护者** | 维护 marketplace 结构，发版，加 skill，同步 `_shared` 到 3 个变体 |
| **销售 / CS 演示者** | 客户面前演示"我们的平台支持 AI 直接驱动"——可视化 9 个模块的串联编排 |
| **速查者** | 临时查个术语 / 缩写 / 工具名 / scope 含义 |

## 一句话价值主张

> Smart TPM Plugins = **把 Smart TPM 后端 9 个模块（datasets / marks / detect_flows / algorithms / test_tasks / detect_records / 链路追踪 / 元数据反查 / 配置查询）共 61 个业务 tool，通过 MCP + OAuth 暴露给本地 AI，配 10 个业务 skill 把高频跨页操作压成一句自然语言。**

## 关键事实

- **11 个 skill** = 1 百科（本文件所在 skill）+ 4 文档型 + 6 任务型（详见 [04-key-concepts.md](./04-key-concepts.md)）
- **9 个模块 / 61 个业务 tool**（最终以后端 `tools/list` 为准；后端 v1.6.1 起，175 部署侧 tool 总数 63 = 61 业务 + 2 文件下载）
- **3 个变体** = local（本机 docker）/ test（内网 121）/ prod（内网 175）
- **写操作 6 个**：
  - `test_tasks_create_from_template` —— 仿模板新建测试任务草稿
  - `marks_set_rough` / `marks_batch_set_rough` —— 单 channel / 批量打粗标（v1.0.10）
  - `marks_add_detail` / `marks_update_detail` / `marks_delete_detail` —— 时间区段细标增删改
  - 全部 service 层自动写 `DetectAnnotationHistory`，createBy/updateBy 是真实 user_id
- **7 个 scope**：`mcp:datasets:{read,write}` / `mcp:detect_flows:read` / `mcp:test_tasks:{read,write}` / `mcp:algorithms:read` / `mcp:detect_records:read`；具体以后端 `<base>/.well-known/oauth-authorization-server` 的 `scopes_supported` 为准
- **鉴权**：OAuth 2.1 + PKCE + DCR，浏览器一次同意即可，无需复制粘贴 token
- **客户端兼容**：Claude Code / Codex / Qwen Code（详见 [11-ai-integration.md](./11-ai-integration.md)）
- **配套姊妹插件**：[SmartTPM_Files_Plugin](https://github.com/bestfunc/SmartTPM_Files_Plugin)（同一 MCP server，OAuth 共享）

## 不在范围

本插件 / 本百科**不覆盖**：

- Smart TPM 检测平台本身的产品功能（前后端、业务流程、UI 设计）—— 那是平台仓库的事
- 算法本身的实现 / 训练 / 评估方法
- 数据集的标注规范与质量控制流程（AI 打标只是辅助工程师手动流程，不替代规范）
- 工位部署 / 硬件运维 / 现场施工

需要这些信息时，回到 Smart TPM 平台仓库的对应文档与 skill（如 `smart-tpm-business-concepts`）。

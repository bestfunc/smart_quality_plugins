# 04 — 核心概念

后续所有文档都建立在这 5 个概念之上。一次性搞清，省得反复猜。

---

## 1. MCP（Model Context Protocol）

**定义**：Anthropic 提出的开放协议，让 AI 客户端（Claude Code / Codex / 第三方 CLI）能以**统一方式**连接任意"上下文源 / 工具服务器"。

**类比**：相当于 LSP（Language Server Protocol）之于 IDE —— 一份协议，多个语言 server 都能挂。MCP 之于 AI，一份协议，多个工具 server 都能挂。

**本仓库里的角色**：
- 协议层 = MCP（由 Anthropic 定义，本仓库不动）
- Server 实现 = Smart TPM 后端 `/api/v2/mcp` 路由（**不在**本仓库，在 `smart_tpm_api` 仓库）
- Client = Claude Code / Codex / Qwen Code（用户机器上）
- 本仓库 = **客户端配置 + skill 文档** 的载体

**Transport**：本插件用 **HTTP transport**（不是 stdio），所以客户端不需要起本地 server 进程，直接连后端 HTTP endpoint。

---

## 2. Tool

**定义**：MCP server 暴露给 AI 的一个可调用函数。每个 tool 有名字 / 参数 schema / 返回 schema / 文档字符串，AI 根据用户意图选 tool 并填参数。

**示例**：本插件里 `datasets_query_samples(datasetId, label='NG', limit=20)` 就是一个 tool。

**计数**：本插件目前 **9 个模块共 61 个业务 tool**（最终以后端 `tools/list` 为准；175 部署侧总 63 = 61 业务 + 2 文件下载走姊妹插件）：

| 模块 | tool 数 | 典型 tool |
|---|---|---|
| 数据集 & 标注（读） | 7 | `datasets_list/get/query_samples/get_annotations`，`datasets_create_draft/import_items/export_manifest`（1 次拿全量 manifest 替代 N 次 get_annotations） |
| 标记系统（读 + 写） | 9 | 读：`project_marks_list/rough_marks_list/detail_marks_list/annotation_history_list`<br>写：`marks_set_rough/batch_set_rough/add_detail/update_detail/delete_detail`（v1.0.10） |
| 检测流程 | 5 | `detect_flows_list/get/list_nodes`，`algorithm_flows_list/get` |
| 算法 | 5 | `algorithms_list/get`，`algorithm_params_search` 等 |
| 测试任务 & 测试记录 | 5 | `test_tasks_list/get/create_from_template`，`test_records_list/get` |
| 生产检测记录 | 3 | `detect_records_list/get/ping`，`detect_points_list`，`detect_stations_list` |
| 链路追踪（v1.0.10） | 9 | `detect_records_trace`（channel_id 一键拿 3 层链路）+ `detect_logs_executions/stages/nodes` + `detect_flow_executions_*` |
| 元数据反查 | 13 | `products/devices/assembly_lines/projects/workstations_list/users_get/detection_dict_list/model_artifacts_*`（3 级树） |
| 配置查询 | 3 | `station_channel_list/detect_station_stages_list/detect_versions_list` |
| 杂项 | 2 | `audio_ai_req_logs_list`（AI 引擎调用快照）、`project_products_list` |

完整名单以各 skill 的 `allowed-tools` frontmatter 为准（Smart TPM 后端 `tools/list` 是最终真相）。

---

## 3. Skill

**定义**：Claude Code 端的一个 markdown 文件（带 frontmatter），描述"什么场景下该做什么、可用哪些 tool、推荐的对话流程"。AI 在对话中根据 `description` 字段自动匹配相关 skill。

**Skill ≠ Tool**：
- Tool 是后端的能力（远程函数）
- Skill 是前端的剧本（让 AI 知道怎么编排 tool）
- 一个 skill 可以引用多个 tool；一个 tool 可被多个 skill 引用

**Skill 的 frontmatter 关键字段**：

```yaml
---
name: <skill-id>           # 全局唯一
description: <用户说什么时该触发>
allowed-tools: a, b, c     # 限定本 skill 内 AI 只能调这些 tool
---
```

**本仓库的 11 个 skill**（详见 [02-architecture.md](./02-architecture.md) 仓库布局）：

| # | Skill | 类型 |
|---|---|---|
| 1 | `about-smart-tpm-mcp` | 百科（user-invocable，本文件所在 skill） |
| 2 | `quickstart-smart-tpm-mcp` | Onboarding |
| 3 | `smart-tpm-business-concepts` | Reference |
| 4 | `dataset-fields-reference` | Reference |
| 5 | `detect-flow-node-reference` | Reference |
| 6 | `sample-dataset-extraction` | Task |
| 7 | `locate-test-record-algorithm-chain` | Task |
| 8 | `create-test-task-from-template` | Task（写） |
| 9 | `reproduce-customer-error` | Task |
| 10 | `query-detect-records` | Task |
| 11 | `mark-samples-with-ai` | Task（写）⚠️ v1.0.10 新加，要 `mcp:datasets:write` |

---

## 4. Scope（OAuth scope）

**定义**：OAuth 授权时用户勾选的细粒度权限单位。本插件按"模块 × 读写"切分为 **7 个 scope**：

| Scope | 含义 |
|---|---|
| `mcp:datasets:read` | 看数据集列表 / 详情 / 抽样本 / 看标注 / 项目标记字典 |
| `mcp:datasets:write` ⚠️ | 创建数据集草稿 / 导入样本 / **AI 打粗标 + 细标**（v1.0.10 起 `marks_set_rough` / `batch_set_rough` / `add_detail` / `update_detail` / `delete_detail` 共 5 写 tool） |
| `mcp:detect_flows:read` | 看检测流程 / 节点 / 执行记录 / 算法流 |
| `mcp:test_tasks:read` | 看测试任务 / 测试记录 |
| `mcp:test_tasks:write` | 基于模板新建测试任务草稿 |
| `mcp:algorithms:read` | 看算法 / 算法流 / 参数库 |
| `mcp:detect_records:read` | 看生产检测记录（工位分库） |

**注意**：
- `detect_flows` / `algorithms` 当前**不开 `:write`** —— 故意的，避免 AI 改坏生产配置
- `datasets:write` 自 v1.0.10 起**含义扩展**：除原"创数据集草稿 / 导入样本"外加 5 个 `marks_*` 打标写 tool。**老用户用 marks 前需要重新 OAuth 勾这个 scope**（之前没相关 tool，大概率没勾）
- 文件下载相关 tool（`files_presign_download` 等）由姊妹插件 [SmartTPM_Files_Plugin](https://github.com/bestfunc/SmartTPM_Files_Plugin) 提供，scope 由该插件管
- 用户每次授权可以**只勾自己需要的 scope**，未勾的 scope 对应的 tool 在 `tools/list` 里被过滤掉
- 具体 scope 名以后端 `<base>/.well-known/oauth-authorization-server` 的 `scopes_supported` 为准

---

## 5. OAuth 2.1 + PKCE + DCR

**定义**：本插件鉴权用的三件套：

| 缩写 | 全称 | 在本插件里做啥 |
|---|---|---|
| **OAuth 2.1** | Open Authorization 2.1 | 标准授权框架。用户跳浏览器登录 + 勾 scope + 回调拿 token |
| **PKCE** | Proof Key for Code Exchange (RFC 7636) | 防止授权码被中间人窃取。客户端生成一次性 `code_verifier`，授权时只带 `code_challenge`，回调换 token 时再用 verifier 证明身份 |
| **DCR** | Dynamic Client Registration (RFC 7591) | 客户端不需要预先在后端注册 `client_id`，第一次连接时自动注册。新用户装插件零配置 |

**用户视角的流程**：

```
1. /plugin install smart-tpm-prod
2. Claude Code 自动跳浏览器，打开 Smart TPM 登录页
3. 用户登录（公司 SSO 或账号密码）
4. 看到 scope 勾选页 → 勾选要给 AI 的权限 → 点同意
5. 浏览器自动跳回，弹窗显示"授权成功，可关闭"
6. Claude Code 客户端拿到 access_token + refresh_token
7. 之后所有 MCP 请求自动带 token；过期自动 refresh
```

**用户不需要做**：
- 复制粘贴 token
- 手动配 `client_id` / `client_secret`
- 改任何配置文件

**撤销**：Smart TPM Web → 个人设置 → 已连接的应用 → 撤销。撤销后客户端下次请求会失败，需要重新走授权流程。

---

## 概念关系图

```
用户       skill          tool          scope          server
 │          │              │              │              │
 └─说话→ AI 选 skill ─调→ tool ─检查→ scope ─放行→ MCP server
                                       ↑
                                  OAuth 授权时勾选
```

简言之：**用户说人话 → AI 用 skill 当剧本 → skill 限定可用 tool → tool 需对应 scope → scope 在 OAuth 时由用户授予**。

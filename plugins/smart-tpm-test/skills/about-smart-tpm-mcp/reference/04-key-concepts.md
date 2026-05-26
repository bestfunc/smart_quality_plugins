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

**计数**：本插件目前约 20 个 tool，分布在 5 个模块（最终以后端 `tools/list` 为准）：

| 模块前缀 | 典型 tool |
|---|---|
| `datasets_*` | `_list` / `_get` / `_query_samples` / `_get_annotations` / `_create_draft` / `_import_samples` |
| `detect_flows_*` | `_list` / `_get` / `_list_nodes` |
| `test_tasks_*` | `_list` / `_get` / `_create_from_template` / `test_records_get` |
| `algorithms_*` | `_list` / `_get` / `algorithm_flows_get` |
| `detect_records_*` | `_ping` / `_list` / `_get` / `detect_stations_list` / `files_presign_download` |

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

**本仓库的 9 个 skill**（详见 [02-architecture.md](./02-architecture.md) 仓库布局）：

| # | Skill | 类型 |
|---|---|---|
| 1 | `quickstart-smart-tpm-mcp` | Onboarding |
| 2 | `smart-tpm-business-concepts` | Reference |
| 3 | `dataset-fields-reference` | Reference |
| 4 | `detect-flow-node-reference` | Reference |
| 5 | `sample-dataset-extraction` | Task |
| 6 | `locate-test-record-algorithm-chain` | Task |
| 7 | `create-test-task-from-template` | Task |
| 8 | `reproduce-customer-error` | Task |
| 9 | `query-detect-records` | Task |

（第 10 个 `about-smart-tpm-mcp` 是本百科，不计入业务 skill。）

---

## 4. Scope（OAuth scope）

**定义**：OAuth 授权时用户勾选的细粒度权限单位。本插件按"模块 × 读写"切分为 6 个 scope：

| Scope | 含义 |
|---|---|
| `mcp:datasets:read` | 看数据集列表 / 详情 / 抽样本 / 看标注 |
| `mcp:datasets:write` | 创建数据集草稿 / 导入样本 / **AI 打粗标 + 细标**(v1.0.10 起,`marks_*` 5 个写 tool) |
| `mcp:detect_flows:read` | 看检测流程 / 节点 / 执行记录 |
| `mcp:test_tasks:read` | 看测试任务 / 测试记录 |
| `mcp:test_tasks:write` | 基于模板新建测试任务草稿 |
| `mcp:algorithms:read` | 看算法 / 算法流 / 参数库 |
| `mcp:detect_records:*` | 看生产检测记录 / 工位 / 文件下载（1.0.4 新加；准确 scope 名以后端 well-known 为准） |

**注意**：
- `detect_flows` 当前**不开 `:write`** —— 故意的，避免 AI 改坏生产配置
- `detect_records` 模块（1.0.4 新加）的具体 scope 名以后端 `<base>/.well-known/oauth-authorization-server` 的 `scopes_supported` 为准；README 截至 1.0.0 的 6-scope 表已过时
- 用户每次授权可以**只勾自己需要的 scope**，未勾的 scope 对应的 tool 在 `tools/list` 里被过滤掉

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

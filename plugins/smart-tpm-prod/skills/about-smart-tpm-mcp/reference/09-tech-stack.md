# 09 — 技术栈

## 定义

本仓库是个**零编译、零运行时**的纯静态资源仓库 —— 只有 markdown 和 JSON。所有"逻辑"都发生在两端：

- **客户端**（Claude Code / Codex / Qwen Code）解析 `plugin.json` 与 skill frontmatter
- **服务端**（Smart TPM 后端）实现 MCP server + OAuth provider

本仓库的"技术栈"本质上是一份**契约清单**：声明给两边看的合同。

## 组成

| 文件类型 | 数量 | 作用 |
|---|---|---|
| `marketplace.json` | 1 | 顶层 marketplace 声明（列出 3 个变体） |
| `plugin.json` | 3 | 每个变体一份；声明 MCP server URL + 元信息 |
| `SKILL.md` | 11 | 10 个业务 skill + 1 本百科 |
| `reference/*.md` `scenarios/*.md` 等 | 本百科子文件 | 本百科的事实层 / 场景层 / 话术层 / 术语层 |
| `README.md` `CHANGELOG.md` | 2 | 仓库门面与版本日志 |

**无构建系统**：没有 `package.json` / `pyproject.toml` / `Makefile`。`git push` 即发布。

## 客户端契约（Claude Code plugin spec）

本仓库遵循 Claude Code 的 plugin 与 marketplace 规范：

- 顶层有 `.claude-plugin/marketplace.json` → 该仓库被识别为 marketplace
- 每个变体目录下 `.claude-plugin/plugin.json` → 该子目录被识别为 plugin
- 变体目录下 `skills/<skill-id>/SKILL.md` → 自动注册为 skill
- `SKILL.md` frontmatter 字段被解析为元数据：
  - `name` — skill 标识
  - `description` — AI 决定何时触发该 skill 的依据
  - `allowed-tools` — 限定 AI 在本 skill 内能调的 MCP tool 子集
  - `user-invocable: true` — 允许用户用 `/skill <name>` 直接调起（本百科用此字段）

详细规范以 Claude Code 官方文档为准。

## 服务端契约（Smart TPM 后端）

后端需要满足：

| 端点 | 用途 |
|---|---|
| `<base>/api/v2/mcp` | MCP HTTP transport endpoint，承载 JSON-RPC（`initialize` / `tools/list` / `tools/call`） |
| `<base>/.well-known/oauth-authorization-server` | OAuth metadata 发现端点，告诉客户端授权 / token / 注册 URL |
| `<base>/oauth/register` | DCR 端点，客户端首次连接时自动注册 |
| `<base>/oauth/authorize` | 跳浏览器的授权页 |
| `<base>/oauth/token` | 用授权码换 access_token / refresh_token |

本契约由 Smart TPM 后端实现，本仓库不动。

## 兼容客户端

| 客户端 | 兼容性 | 备注 |
|---|---|---|
| **Claude Code** | ✅ 一等公民 | 本仓库的 plugin spec 主要遵循其规范 |
| **OpenAI Codex CLI** | ✅ 兼容 | 通过 MCP server 通用接口接入 |
| **Qwen Code CLI** | ✅ 兼容（1.0.2 起） | 需 `httpUrl` 字段（详见 [11-ai-integration.md](./11-ai-integration.md)） |
| 其他 MCP 客户端 | 视实现而定 | 只要支持 HTTP transport + OAuth 2.1，理论可接 |

## 约束清单（给维护者）

写代码 / 改文件时要遵守：

1. **不引入构建依赖**。仓库永远保持"clone 即可用"，不加 `npm install` / `pip install` 这种前置步骤。
2. **三变体的 `version` 字段必须同步**。bump 一个就 bump 全部。
3. **`mcpServers.smart-tpm.key` 必须固定为 `smart-tpm`**。改了会让 skill 里的 tool 引用断链。
4. **每个 skill 的 `allowed-tools` 必须是后端 `tools/list` 的真子集**。后端没注册的 tool 写进 frontmatter，AI 调用直接失败 —— 改 frontmatter 后必须 e2e 实测验证。
5. **`_shared/skills/<x>/` 改完后，必须物理复制到 3 个变体目录**。漏一个变体就会版本漂移。
6. **JSON 不要写注释**。`plugin.json` / `marketplace.json` 是严格 JSON，加 `//` 注释会被 Claude Code 拒绝解析。
7. **跨文档引用一律相对路径**。方便在 GitHub 网页渲染时也能点。

## Schema 真相源

| 想知道什么 | 看哪 |
|---|---|
| 当前 plugin spec 字段 | Claude Code 官方文档 + 本仓库 `plugin.json` 实例 |
| 当前后端 tool 名单 | 直接调 MCP `tools/list`（最权威） |
| 当前 OAuth scope 名单 | `<base>/.well-known/oauth-authorization-server` 的 `scopes_supported` |
| 历史变更 | `git log` + [CHANGELOG.md](../../../../../CHANGELOG.md) |

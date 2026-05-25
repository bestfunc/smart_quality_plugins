# 11 — AI 客户端集成

## 定义

本插件用 **MCP HTTP transport + OAuth 2.1**，理论上任何符合 MCP 规范的客户端都能接。实操中本仓库主要在 3 个客户端上验证过：Claude Code / OpenAI Codex / Qwen Code。本文记录每个客户端的接入差异与已知坑。

## 兼容总表

| 客户端 | 是否需特殊处理 | 当前状态 |
|---|---|---|
| **Claude Code** | 否 | ✅ 一等公民。`/plugin marketplace add bestfunc/smart_quality_plugins` 即装即用 |
| **OpenAI Codex CLI** | 否 | ✅ 走 MCP 通用接口；`plugin.json` 的 `mcpServers` 块被 Codex 直接读取 |
| **Qwen Code CLI** | 是 | ✅ 需 `httpUrl` 字段补丁（1.0.2 起内建） |
| 其他 MCP 客户端 | 视实现 | ⚠️ 未系统验证 |

## Claude Code

**安装路径**：

```
> /plugin marketplace add bestfunc/smart_quality_plugins
> /plugin install smart-tpm-prod
[Claude Code 跳浏览器走 OAuth，授权完成后注册 skill]
```

**注意**：
- Claude Code 不同版本对 `marketplace.json` 的 `plugins[].source` 字段格式要求不同。本仓库统一用相对路径字符串（`"./plugins/smart-tpm-prod"`），与新版兼容。早期版本曾踩过 `"."` / `"./"` 不被接受、需改成 `{"source": "github", "repo": "..."}` 对象形式的坑（已在 1.0.x 修过）。
- 装完后用 `/mcp` 查看 `smart-tpm` 服务器状态；`/skill` 查看本插件注册的所有 skill。

## OpenAI Codex CLI

**接入方式**：Codex 读取 `plugin.json` 里的 `mcpServers` 节并直接拉起 MCP 连接。本插件用 HTTP transport，Codex 端无需起子进程。

**注意**：
- OAuth 浏览器流程与 Claude Code 一致。
- Codex 对 skill frontmatter 的处理可能与 Claude Code 略不同（如 `allowed-tools` 是否强制裁剪）；以 Codex 当前版本文档为准。
- 本插件未针对 Codex 做专门兼容补丁 —— 因为它走的是 MCP 通用契约。

## Qwen Code CLI

**已知坑（1.0.2 起内建修复）**：

Qwen Code 对 MCP `mcpServers` 配置块的解析，会**误把缺 `httpUrl` 字段的 server 识别为 SSE transport**（Server-Sent Events），从而走错连接路径，导致连接失败 / tool 不可见。

修复办法是**同时写 `url` 和 `httpUrl` 两个字段且值相同**：

```json
{
  "mcpServers": {
    "smart-tpm": {
      "type": "http",
      "url": "http://192.168.2.175:8080/api/v2/mcp",
      "httpUrl": "http://192.168.2.175:8080/api/v2/mcp"
    }
  }
}
```

`url` 是标准字段（Claude Code / Codex 读这个），`httpUrl` 是 Qwen 期望的字段（专门给它读）。两个字段对其他客户端是冗余，但**不冲突**，所以本仓库三变体都一并加上。

## 跨客户端共通的安装前提

不论用哪个客户端，需要满足：

| 前提 | 检查办法 |
|---|---|
| 客户端能访问对应变体的后端 URL | 在终端 `curl <url>` 试一下，至少能拿到 HTTP 响应 |
| 浏览器能打开后端的 OAuth 授权页 | 装完插件第一次连接时会弹浏览器；如未弹，看客户端日志 |
| 该用户在 Smart TPM 平台已有账号 | OAuth 授权页要登录，没账号过不去 |
| 该账号已被授予所需 scope | 如未授予，授权页勾选时对应选项会灰掉 |

## 切换客户端时

如果同事原来用 Claude Code，现在改用 Codex（或反之）：

- **不需要重装插件本身**（plugin 文件是公共的，由 marketplace 提供）
- **需要重新 OAuth 授权**（token 绑客户端的 `client_id`，DCR 注册的 `client_id` 不同客户端不通用）
- skill 缓存可能要清一下（具体命令看各客户端文档）

## 测试新客户端兼容性

要评估某个新 MCP 客户端能否接入本插件，最小验证步骤：

1. 让客户端读 `plugin.json`（或对应配置格式），看是否能正确识别 MCP server
2. 客户端能否走完 OAuth 2.1 + PKCE + DCR 流程（这步最容易出兼容性问题）
3. 调 `tools/list`，能否拿到本插件的全部 tool（约 20 个）
4. 触发其中一个 read tool（如 `datasets_list`），能否拿到返回
5. 加载某个 skill 文件，能否被客户端解析 frontmatter

跑通这 5 步，基本可认定兼容。

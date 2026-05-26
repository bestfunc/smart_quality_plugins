# 03 — 变体对比（local / test / prod）

## 定义

本仓库作为一个 marketplace，发布**三个变体**。三者**代码完全相同（11 个 skill 一模一样）**，唯一差别在 `.claude-plugin/plugin.json` 里 `mcpServers.smart-tpm.url` 指向的后端环境。用户根据自己在哪个环境调试，装对应那一个即可。

## 全表对比

| 维度 | smart-tpm-local | smart-tpm-test | smart-tpm-prod |
|---|---|---|---|
| **后端 URL** | `http://localhost:8000/api/v2/mcp` | `http://192.168.2.121:8080/api/v2/mcp` | `http://192.168.2.175:8080/api/v2/mcp` |
| **目标环境** | 本机 docker 起的 `smart-tpm-api` 容器 | 部门内测环境（192.168.2.121） | 部门生产环境（192.168.2.175） |
| **谁该装** | 后端开发 / 算法调试 / MCP 工具开发 | QA / 数据 / 算法做联调 / 演示验证 | 现场工程师 / CS / 销售面客 |
| **数据真实性** | 自己 seed 的 mock 数据 | 接近真实的内测数据 | 真实生产数据（含客户记录） |
| **写权限风险** | 低（坏了 docker 重启） | 中（影响 QA） | **高**（影响真实运营） |
| **可访问性** | 仅本机 | 仅部门内网 | 仅部门内网 |
| **OAuth 授权页** | localhost:8000 上的同源页面 | 121:8080 的同源页面 | 175:8080 的同源页面 |
| **HTTPS** | 否（本机） | 否（内网） | 否（**故意**走 IP 内网避开自签证书） |

## 为什么 prod 用内网 IP 而不是域名

`smartquality.bestfunc.com` 这个生产域名挂的是**自签证书**。Claude Code / Codex / Qwen 这类客户端对自签证书的处理不一致（部分静默失败、部分抛错），且本插件**客户全在内网**，没必要硬上 HTTPS。

- 1.0.3 之前曾用 `https://smartquality.bestfunc.com/api/v2/mcp` → 部分客户端授权环节就卡住
- 1.0.3 起切到 `http://192.168.2.175:8080/api/v2/mcp` → 全客户端打通

如果未来上公网部署 / 接入正式证书，prod URL 会改回域名形式。

## 三变体能否共存

可以。Claude Code 允许同时安装多个 plugin，只要 `mcpServers` 的 key 不冲突。

但**本仓库三个变体的 MCP server key 全是 `smart-tpm`**（故意的，让 skill 里的 tool 引用能跨变体复用）。所以**同一台机器同时启用 smart-tpm-local 和 smart-tpm-prod，后注册的会覆盖先注册的**。建议同一时间只启用一个变体。

**切环境的轻量方法**（推荐）：改 `.claude/settings.json` 的 `enabledPlugins` 后缀（如把 `smart-tpm-test@smart_quality_plugins` 改成 `smart-tpm-prod@smart_quality_plugins`）→ 重启 Claude Code。无需 uninstall / install。但**注意 OAuth token 是按环境 + DCR client_id 绑定的**，切完通常要重走一次浏览器授权。

## 选哪个

| 你的场景 | 装哪个 |
|---|---|
| 我在本机起了 docker，正在改后端 / 调 MCP tool | `smart-tpm-local` |
| 我要给同事 / 客户演示，但**没必要碰真数据** | `smart-tpm-test` |
| 我是现场工程师 / CS，处理真实工单 | `smart-tpm-prod` |
| 我是销售，要给客户做现场 demo | `smart-tpm-test`（**不**用 prod，避免误操作） |
| 我同时要看测试和生产 | 默认启用 prod；要切 test 时改 `.claude/settings.json` 的 `enabledPlugins` 后缀 → 重启 → 重新 OAuth |

## 版本号策略

三变体的 `version` 字段**始终保持一致**。bump 任何一个版本号都意味着同步 bump 另两个。版本演进见 [CHANGELOG.md](../../../../../CHANGELOG.md)（项目根）与 [08-roadmap.md](./08-roadmap.md)。

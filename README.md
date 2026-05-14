# SmartTPM_Plugins

Smart TPM 检测平台的 Claude Code 插件，通过 MCP（Model Context Protocol）把数据集 / 检测流程 / 测试任务 / 算法 4 个模块的 21 个工具暴露给工程师本地 AI 客户端（Claude Code / Codex / Qwen CLI），用自然语言驱动 AI 完成"查、对照、初稿配置"。

## 安装

```
$ claude-code
> /plugin add bestfunc/SmartTPM_Plugins
✓ Plugin 'smart-tpm' installed
  MCP server 'smart-tpm' needs authorization. Open browser? (Y/n) Y
  [浏览器弹出 Smart TPM 登录页 → 授权页]
  [勾选 datasets / detect_flows / test_tasks / algorithms 的 read scope]
✓ Authorized.
```

授权完成后，所有 Skill 自动可用。

## 鉴权

OAuth 2.1 + PKCE + DCR。**不需要复制粘贴 token**，浏览器一次同意即可。Token 过期会自动 refresh；可在 Smart TPM Web → 个人设置 → 已连接的应用 撤销。

## 可用 Scope

| Scope | 说明 |
|---|---|
| `mcp:datasets:read` | 数据集列表 / 详情 / 抽样本 / 看标注 |
| `mcp:datasets:write` | 创建数据集草稿 / 导入样本 |
| `mcp:detect_flows:read` | 检测流程列表 / 详情 / 节点 / 执行记录 |
| `mcp:test_tasks:read` | 测试任务 / 测试记录 列表 / 详情 |
| `mcp:test_tasks:write` | 基于模板新建测试任务草稿 |
| `mcp:algorithms:read` | 算法 / 算法流 / 参数库 |

> `detect_flows` 首期不开 `:write`（避免 AI 改坏生产配置）。

## Skill 一览（8 个）

| # | Skill | 类型 | 主要工具 |
|---|---|---|---|
| 1 | 快速上手 Smart TPM MCP | Onboarding | datasets_list / detect_flows_list / test_tasks_list / algorithms_list |
| 2 | Smart TPM 业务概念速查 | Reference（纯文档） | — |
| 3 | 数据集字段速查 | Reference | — |
| 4 | 检测流程节点速查 | Reference | — |
| 5 | 抽取数据集样本 | Task | datasets_list / datasets_get / datasets_query_samples / datasets_get_annotations |
| 6 | 定位测试记录算法链路 | Task | test_records_get / detect_flows_get / algorithm_flows_get / algorithms_get |
| 7 | 基于模板新建测试任务 | Task | test_tasks_list / test_tasks_get / test_tasks_create_from_template |
| 8 | 复现客户报错 | Task | 全 *_read |

## 上线 Checklist（仓库维护者）

- [ ] 替换 `plugin.json` 中的 `<smart-tpm-domain>` 为生产域名（必须 https）
- [ ] 确认生产后端 `/api/v2/mcp` 已挂载，`/.well-known/oauth-authorization-server` 可访问
- [ ] 在 `bestfunc` 组织下 push 此仓库为 public
- [ ] 更新 CHANGELOG.md
- [ ] 通知工程师按"安装"步骤接入

## License

Internal use only.

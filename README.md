# SmartTPM_Plugins

Smart TPM 检测平台的 Claude Code plugin **marketplace**，通过 MCP（Model Context Protocol）把 datasets / detect_flows / test_tasks / algorithms / detect_records 5 个模块的工具暴露给工程师本地 AI 客户端（Claude Code / Codex / Qwen Code），用自然语言驱动 AI 完成"查、对照、初稿配置"。

> 本仓库是一个 marketplace，提供 3 个变体（local / test / prod），按你的目标环境装对应那一个。Skill 全部由 `_shared/skills` 物理同步到 3 变体。

## 安装

```
> /plugin marketplace add bestfunc/smart_quality_plugins
> /plugin install smart-tpm-prod         # 或 smart-tpm-test / smart-tpm-local
[浏览器弹出 Smart TPM 登录页 → 授权页 → 勾选所需 scope → 同意]
✓ Authorized.
```

授权完成后，所有 Skill 自动可用。变体选哪个见 [reference/03-edition-comparison.md](plugins/_shared/skills/about-smart-tpm-mcp/reference/03-edition-comparison.md)。

## 鉴权

OAuth 2.1 + PKCE + DCR。**不需要复制粘贴 token**，浏览器一次同意即可。Token 过期会自动 refresh；可在 Smart TPM Web → 个人设置 → 已连接的应用 撤销。

## Scope（按模块 × 读写切分）

| Scope | 说明 |
|---|---|
| `mcp:datasets:read` | 数据集列表 / 详情 / 抽样本 / 看标注 |
| `mcp:datasets:write` | 创建数据集草稿 / 导入样本 |
| `mcp:detect_flows:read` | 检测流程列表 / 详情 / 节点 / 执行记录 |
| `mcp:test_tasks:read` | 测试任务 / 测试记录 列表 / 详情 |
| `mcp:test_tasks:write` | 基于模板新建测试任务草稿 |
| `mcp:algorithms:read` | 算法 / 算法流 / 参数库 |
| `mcp:detect_records:*` | 生产线检测记录 / 工位 / 文件下载（1.0.4 加入） |

> `detect_flows` 当前不开 `:write`（防 AI 改坏生产配置）。具体 scope 名以后端 `<base>/.well-known/oauth-authorization-server` 的 `scopes_supported` 为准。

## Skill 一览（10 个 = 9 业务 + 1 百科）

| # | Skill | 类型 | 主要工具 |
|---|---|---|---|
| 1 | `quickstart-smart-tpm-mcp` | Onboarding | datasets_list / detect_flows_list / test_tasks_list / algorithms_list |
| 2 | `smart-tpm-business-concepts` | Reference（纯文档） | — |
| 3 | `dataset-fields-reference` | Reference | — |
| 4 | `detect-flow-node-reference` | Reference | — |
| 5 | `sample-dataset-extraction` | Task | datasets_list / datasets_get / datasets_query_samples / datasets_get_annotations |
| 6 | `locate-test-record-algorithm-chain` | Task | test_records_get / detect_flows_get / algorithm_flows_get / algorithms_get |
| 7 | `create-test-task-from-template` | Task | test_tasks_list / test_tasks_get / test_tasks_create_from_template |
| 8 | `reproduce-customer-error` | Task | 全 *_read |
| 9 | `query-detect-records` | Task | detect_records_ping / _list / _get / detect_stations_list / files_presign_download |
| 10 | `about-smart-tpm-mcp` | 百科（user-invocable） | — |

## 维护者文档

- 仓库布局与同步规则：[reference/02-architecture.md](plugins/_shared/skills/about-smart-tpm-mcp/reference/02-architecture.md)
- 发版流程：[scenarios/02-maintainer-release-workflow.md](plugins/_shared/skills/about-smart-tpm-mcp/scenarios/02-maintainer-release-workflow.md)
- 常见维护问题：[faq/02-developer-questions.md](plugins/_shared/skills/about-smart-tpm-mcp/faq/02-developer-questions.md)

## License

Internal use only.

# SmartTPM_Plugins

Smart TPM 检测平台的 Claude Code plugin **marketplace** — 通过 MCP(Model Context Protocol)把后端 9 个模块、**61 个业务 tool** 暴露给本地 AI 客户端(Claude Code / Codex / Qwen Code),让工程师用自然语言查检测数据、追溯算法链路、复现客户报错、辅助打标记。

> 文件下载场景(MinIO bucket / 数据集音频 / 模型 artifact)走配套的姊妹插件 [SmartTPM_Files_Plugin](https://github.com/bestfunc/SmartTPM_Files_Plugin),两个 plugin 指向同一个 MCP server 端点,**OAuth 一次浏览器同意即可两个都生效**。

## 安装

```
> /plugin marketplace add bestfunc/smart_quality_plugins
> /plugin install smart-tpm-prod@smart_quality_plugins
[浏览器弹出 Smart TPM 登录页 → 授权页 → 勾选所需 scope → 同意]
✓ Authorized.
```

变体三选一:

| 变体 | URL | 用途 |
|---|---|---|
| `smart-tpm-prod` | `http://192.168.2.175:8080` | **生产**(内网 IP 直连,绕开自签证书) |
| `smart-tpm-test` | `http://192.168.2.121:8080` | **测试**(部门内测环境) |
| `smart-tpm-local` | `http://localhost:8000` | **本地**(本机 docker 容器) |

切环境:改 `.claude/settings.json` 的 `enabledPlugins` 后缀(`-prod` ↔ `-test` ↔ `-local`)→ 重启 Claude Code。

## 鉴权

OAuth 2.1 + PKCE + DCR。**不需要复制粘贴 token**,浏览器一次同意即可。Token 自动 refresh,可在 Smart TPM Web → 个人设置 → 已连接的应用 撤销。

## Scope(7 条,按模块 × 读写切分)

| Scope | 控制的工具 |
|---|---|
| `mcp:datasets:read` | 数据集/样本/标注/项目标记字典 读(`datasets_*`, `marks` 读, `project_marks_list`, `metadata_*` 等) |
| `mcp:datasets:write` ⚠️ | 创建数据集草稿 / 导入样本 / **AI 打粗标 + 细标**(`marks_set_rough`, `marks_batch_set_rough`, `marks_add_detail` 等 5 写 tool,v1.0.10 起) |
| `mcp:detect_flows:read` | 检测流程定义 / 节点 / 执行链路(`detect_flows_*`, `algorithm_flows_*`, `detect_flow_executions_*`, `detect_logs_*` 等) |
| `mcp:test_tasks:read` | 测试任务 / 测试记录 列表 / 详情 |
| `mcp:test_tasks:write` | 基于历史任务模板新建测试任务草稿 |
| `mcp:algorithms:read` | 算法 / 算法流 / 算法参数库 |
| `mcp:detect_records:read` | 生产线 detect_channel 检测记录(工位分库) |

> `detect_flows` 不开 `:write`(防 AI 改坏生产配置);`algorithms` 同理。
> 完整 scope 列表以后端 `<base>/.well-known/oauth-authorization-server` 的 `scopes_supported` 为准。

## MCP Tool 一览(按模块切,共 61 个)

| 模块 | tool 数 | 说明 |
|---|---|---|
| 数据集 & 标注 | 7 | `datasets_list/get`, `datasets_query_samples`, `datasets_get_annotations`, `datasets_create_draft`, `datasets_import_items`, `datasets_export_manifest`(1 次拿全量 manifest,替代 N 次单条调用) |
| **标记系统**(读 + 写) | 9 | 读:`project_marks_list`, `rough_marks_list`, `detail_marks_list`, `annotation_history_list`<br>**写**(v1.0.10):`marks_set_rough`, `marks_batch_set_rough`, `marks_add_detail`, `marks_update_detail`, `marks_delete_detail` |
| 检测流程 | 5 | `detect_flows_list/get/list_nodes`, `algorithm_flows_*` |
| 算法 | 5 | `algorithms_list/get`, `algorithm_params_search` 等 |
| 测试任务 & 测试记录 | 5 | `test_tasks_list/get/create_from_template`, `test_records_list/get` |
| 生产检测记录 | 3 | `detect_records_list/get/ping`, `detect_points_list`, `detect_stations_list` |
| 链路追踪(v1.0.10) | 9 | `detect_records_trace`(channel_id 一键拿 3 层链路) + `detect_logs_executions/stages/nodes_*` + `detect_flow_executions_*` |
| 元数据反查 | 13 | `products`, `devices`, `assembly_lines`, `projects`, `workstations_list`, `users_get`, `detection_dict_list`, `model_artifacts_*`(3 级树) |
| 配置查询 | 3 | `station_channel_list`, `detect_station_stages_list`, `detect_versions_list` |
| 杂项 | 2 | `audio_ai_req_logs_list`(算法引擎调用快照), `project_products_list` |

## Skill 一览(16 个 = 6 百科 + 4 文档 + 6 任务)

| # | Skill | 类型 | 主要工具 |
|---|---|---|---|
| 1 | `about-smart-tpm-mcp` | 百科(user-invocable) | — |
| 2 | `quickstart-smart-tpm-mcp` | Onboarding | `datasets_list`, `detect_flows_list`, `test_tasks_list`, `algorithms_list` |
| 3 | `smart-tpm-business-concepts` | Reference(纯文档) | — |
| 4 | `dataset-fields-reference` | Reference(纯文档) | — |
| 5 | `detect-flow-node-reference` | Reference(纯文档) | — |
| 6 | `sample-dataset-extraction` | Task | `datasets_query_samples`, `datasets_get_annotations`, `datasets_export_manifest` |
| 7 | `locate-test-record-algorithm-chain` | Task | `test_records_get`, `detect_records_trace`, `detect_logs_*`, `algorithm_flows_get`, `algorithms_get` |
| 8 | `create-test-task-from-template` | Task(写) | `test_tasks_create_from_template` |
| 9 | `reproduce-customer-error` | Task | 全 *_read + `annotation_history_list` + `detect_records_trace` + `audio_ai_req_logs_list` |
| 10 | `query-detect-records` | Task | `detect_records_*`, `detect_stations_list`, `files_presign_download` |
| 11 | **`mark-samples-with-ai`** ⚠️ | Task(写) | `marks_set_rough/batch_set_rough/add_detail/update_detail/delete_detail` — **要 `mcp:datasets:write`** |
| 12 | `about-audio-quality` | 百科(user-invocable) | — (纯文档:音频质检后端 ai_api_server_v2 的架构/DAG引擎/算法模型/集群部署百科) |
| 13 | `about-net-audio` | 百科(user-invocable) | — (纯文档:net_audio 联网录音系统的双进程架构/A-B-C-D 通道/灵敏度调校/上游算法对接/现场部署百科) |
| 14 | `about-sync-data` | 百科(user-invocable) | — (纯文档:多客户端数据同步系统百科) |
| 15 | `about-smartplc` | 百科(user-invocable) | — (纯文档:SmartPLC / BestPLC 产线声学质检系统百科 — 多协议 PLC 中控 + 录音盒子采音 + AI 判 OK/NG + 回写控流转;按开发/销售/现场实施/速查四类受众分流) |
| 16 | `about-smart-quality` | 百科(user-invocable) | — (纯文档:SmartQuality 智能质量检测平台产品全貌与离职交接百科 — 覆盖微信服务/数据同步(服务端+客户端)/采集端/Web平台/移动套装等交接系统;含架构/核心概念/技术栈/生态导航/交接对照表) |

## 写操作安全 / 审计

- 所有 `marks_*` 写 tool 自动写 `DetectAnnotationHistory` 审计表,`createBy`/`updateBy` 是被授权的真实用户 user_id(不是字面值 `"MCP"`)
- 反查:`annotation_history_list(channel_id=..., date_from=...)` 看 AI 哪次会话改了什么
- 撤销:`marks_delete_detail`(软删,history 留底);粗标重新 `marks_set_rough(mark_code="")` 清掉
- `mark-samples-with-ai` skill 内置 SOP:必须先 `project_marks_list` 拿字典再用 markCode,默认 `overwrite=false` 不动冲突,批量上限 200 条/次

## 维护者文档

- 仓库布局与 _shared 同步规则:[reference/02-architecture.md](plugins/_shared/skills/about-smart-tpm-mcp/reference/02-architecture.md)
- 发版流程:[scenarios/02-maintainer-release-workflow.md](plugins/_shared/skills/about-smart-tpm-mcp/scenarios/02-maintainer-release-workflow.md)
- 常见维护问题:[faq/02-developer-questions.md](plugins/_shared/skills/about-smart-tpm-mcp/faq/02-developer-questions.md)
- 完整版本记录:[CHANGELOG.md](CHANGELOG.md)

## 当前版本

`v1.0.14`(并入 `about-sync-data` 多客户端数据同步系统百科 + `about-smartplc` SmartPLC 产线声学质检系统百科,两个新百科 skill 同版本号入册,纯文档,后端 tool 无变化)

## License

Internal use only.

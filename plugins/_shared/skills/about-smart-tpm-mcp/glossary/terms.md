# 术语 / 缩写表

> 本仓库与 Smart TPM MCP 集成相关的全部专有名词与缩写。按字母序排列。

## 缩写

| 缩写 | 全称 | 说明 / 链接 |
|---|---|---|
| **API** | Application Programming Interface | 应用编程接口。本仓库特指 Smart TPM 后端 HTTP 接口（`/api/v2/mcp`） |
| **CLI** | Command Line Interface | 命令行客户端。本插件支持的 AI CLI：Claude Code / Codex / Qwen Code |
| **CS** | Customer Success | 客户成功岗位 |
| **DCR** | Dynamic Client Registration | RFC 7591 动态客户端注册。客户端首次连接时自动向服务端注册 `client_id`，免去预配置。详见 [reference/04-key-concepts.md](../reference/04-key-concepts.md) |
| **HTTP** | HyperText Transfer Protocol | 本插件 MCP transport 用 HTTP（非 stdio） |
| **JSON-RPC** | JSON Remote Procedure Call | MCP 协议底层 wire format |
| **MCP** | Model Context Protocol | Anthropic 提出的开放协议，让 AI 客户端连接工具服务器。详见 [reference/04-key-concepts.md](../reference/04-key-concepts.md) |
| **OAuth** | Open Authorization | 本插件用 OAuth 2.1 + PKCE + DCR 三件套做鉴权 |
| **PKCE** | Proof Key for Code Exchange | RFC 7636 防授权码窃取的 OAuth 扩展。详见 [reference/04-key-concepts.md](../reference/04-key-concepts.md) |
| **SSE** | Server-Sent Events | 一种服务端推送 transport。本插件**不用** SSE，但 Qwen Code 误识别问题与此相关（详见 [reference/11-ai-integration.md](../reference/11-ai-integration.md)） |
| **SSO** | Single Sign-On | 单点登录。Smart TPM 可对接公司 SSO，OAuth 授权页可能跳到 SSO 完成登录 |
| **TPM** | Total Productive Maintenance / Smart TPM | "Smart TPM" 是本平台产品名（取 Total Productive Maintenance 之意） |
| **URL** | Uniform Resource Locator | 三变体各自 `mcpServers.smart-tpm.url` 指向不同 |

## 概念

| 术语 | 说明 |
|---|---|
| **algorithm（算法）** | Smart TPM 平台的核心业务对象。本插件 `algorithms_*` tool 系列只读访问算法库 / 算法流 / 参数模板 |
| **algorithm_flow（算法流）** | 算法的有序组合。`algorithm_flows_get` 是相关 tool |
| **allowed-tools** | Skill frontmatter 字段，限定 AI 在本 skill 内能调的 MCP tool 子集 |
| **annotation_history（标注历史 / DetectAnnotationHistory 表）** | 所有打标改动落审计的业务表;`createBy`/`updateBy` 是真实 user_id。tool `annotation_history_list` 反查"谁/AI 哪次会话改了哪个 channel" |
| **audit log** | `mcp_audit_log` 表。所有 MCP 调用落日志，超管在 Web → 系统设置 → MCP 审计 可查 |
| **bestfunc** | 本仓库所在 GitHub 组织 / 公司账号名 |
| **Claude Code** | Anthropic 官方 AI 编程 CLI（本插件主要目标客户端，OAuth + MCP 支持最完整） |
| **Codex** | OpenAI 推出的 AI 编程 CLI，支持 MCP（本插件兼容） |
| **Qwen Code** | 阿里基于 Codex 改的中文 AI 编程 CLI；对 SSE transport 有误识别问题，需 `httpUrl` 字段绕过（1.0.2 起兼容） |
| **dataset（数据集）** | Smart TPM 平台核心业务对象。本插件 `datasets_*` tool 系列提供读 + 受控写 |
| **detect_channel（检测通道）** | 生产线上一个具体的检测点。`detect_records` 模块按 channel 拉数据 |
| **detect_flow（检测流程）** | 一系列检测节点组成的流水线。本插件 `detect_flows_*` tool 只读访问 |
| **detect_flow_executions（流程执行实例）** | 一次检测流程跑出来的执行实例(包含每个 stage / node 的中间产物)。v1.0.9 起接入,tool `detect_flow_executions_*` |
| **detect_logs**（算法节点日志） | 算法节点级 input / output 快照。v1.0.9 起 `detect_logs_executions/stages/nodes_*` 一组 3 个 tool 拿算法工程师视角的完整链路 |
| **detect_records（检测记录）** | 生产线实时检测产生的记录。1.0.4 起接入，按 workstation 物理分库 |
| **detect_records_trace** | v1.0.9 起的"杀手 tool":给 channel_id 一键拿测试记录 → 流程执行 → 算法日志 3 层链路。原来要 4 个 tool 串起来,现在 1 个搞定 |
| **detect_stations / workstation（工位）** | 物理产线工位。每个 workstation 自己一个 MySQL 实例（数据物理隔离） |
| **edition / 变体** | 本 marketplace 提供的 3 个 plugin 变体：`smart-tpm-local` / `smart-tpm-test` / `smart-tpm-prod` |
| **frontmatter** | Markdown 文件顶部用 `---` 包裹的 YAML 元数据。SKILL.md 用它声明 name / description / allowed-tools |
| **httpUrl** | `mcpServers.smart-tpm.httpUrl`。与 `url` 同值，专为兼容 Qwen Code 而加（1.0.2 起） |
| **marketplace** | Claude Code 的 plugin 仓库聚合形态。本仓库根 `.claude-plugin/marketplace.json` 声明 3 个 plugin |
| **markCode / project_marks（项目标记字典）** | 每个项目自己定义的合法标签集(如 "OK" / "NG" / "scratch")。tool `project_marks_list` 拿字典,AI 打标前**必须先查再用**(不让 AI 编 markCode) |
| **marks_* 写 tool** | v1.0.10 新加的 5 个打标写 tool: `marks_set_rough` / `marks_batch_set_rough` (粗标,channel 级) + `marks_add_detail` / `marks_update_detail` / `marks_delete_detail` (细标,时间区段)。要 `mcp:datasets:write` scope |
| **model_artifacts** | 算法模型 artifact 树(3 级)。`model_artifacts_*` 一组元数据反查 tool |
| **org（组织）** | Smart TPM 平台租户单位。service 层按 org 隔离数据 |
| **plugin** | Claude Code 的可装载单元。本仓库每个变体（local/test/prod）= 一个 plugin |
| **plugin.json** | Plugin 元信息文件，位于每个变体的 `.claude-plugin/plugin.json` |
| **粗标(rough mark)** | channel 级整体打标(整段录音/整个样本 = 一个 markCode)。tool `marks_set_rough` / `marks_batch_set_rough` |
| **细标(detail mark)** | 时间区段级打标(channel 内 12.5-18.3s 段标成 NG)。tool `marks_add_detail` 等 |
| **prod / production** | 生产环境变体（`smart-tpm-prod`），后端 URL `http://192.168.2.175:8080` |
| **refresh token** | OAuth token 过期时用 refresh_token 换新 access_token，对用户无感 |
| **scope** | OAuth 授权的细粒度权限单位。本插件按模块 × 读写切分（详见 [reference/04-key-concepts.md](../reference/04-key-concepts.md)，最终以后端 well-known 为准） |
| **service layer** | Smart TPM 后端的业务隔离层，强制按 org + workstation 过滤数据 |
| **skill** | Claude Code 端的 markdown 剧本（带 frontmatter）。本仓库当前 10 个业务 skill + 1 本百科 |
| **Skill 类型：Onboarding** | 第一次使用时引导，本仓库的 `quickstart-smart-tpm-mcp` |
| **Skill 类型：Reference** | 纯文档型，本仓库的 `smart-tpm-business-concepts` / `dataset-fields-reference` / `detect-flow-node-reference` |
| **Skill 类型：Task** | 任务编排型，本仓库的 6 个 task skill |
| **Smart TPM** | 本平台产品名 |
| **SmartTPM_Files_Plugin** | 姊妹插件,文件下载(MinIO bucket / 音频 / 模型 artifact)走它。指向同一个 MCP server,与本仓库 OAuth 共享 |
| **smart_quality_plugins** | 本仓库名（GitHub repo）；本插件 marketplace 内部标识 |
| **smart_tpm_web_v2** | Smart TPM 后端仓库名。MCP server 实现在此(FastAPI 路由 + tool 注册)。当前 v1.6.1 配套 plugin v1.0.10 |
| **smart-tpm-local** | 变体之一，连本机 docker（localhost:8000） |
| **smart-tpm-test** | 变体之一，连内网测试环境（192.168.2.121:8080） |
| **smart-tpm-prod** | 变体之一，连内网生产环境（192.168.2.175:8080） |
| **test_record（测试记录）** | 一次测试任务下的具体一条记录。复盘客户报错的核心入口 |
| **test_task（测试任务）** | 一组测试记录的执行单元 |
| **tool** | MCP server 暴露给 AI 的可调用函数。本插件 v1.0.10 共 61 个业务 tool 跨 9 个模块(最终以后端 `tools/list` 为准) |
| **transport** | MCP 协议底层传输方式。本插件用 HTTP（非 stdio / SSE） |
| **user-invocable** | Skill frontmatter 字段。设为 `true` 时用户可 `/skill <name>` 直接调起 |
| **workstation_id** | 工位标识。`detect_records_*` 系列 tool 必填第一参数（数据按 workstation 物理分库） |

## 命名约定

| 模式 | 含义 |
|---|---|
| `<module>_list` | 模块列表查询 |
| `<module>_get` | 模块单实例详情查询 |
| `<module>_<verb>` | 模块特定动作（如 `datasets_query_samples`） |
| `*_create_from_template` | 基于模板新建（本插件唯一的写入口） |
| `*_get_annotations` | 拿标注信息 |
| `*_presign_download` | 拿预签名下载 URL（用于音频 / 图片等大文件） |
| `*_ping` | 健康检查 / 联通性测试 |

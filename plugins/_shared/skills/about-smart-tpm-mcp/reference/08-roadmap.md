# 08 — 路线图

> 本节内容均为**计划 / 在调研**，不构成承诺。实际发布以 [CHANGELOG.md](../../../../../CHANGELOG.md) 与 marketplace 上发布的最新 version 为准。

## 已发布（事实层）

| 版本 | 关键动作 |
|---|---|
| 1.0.0 | 首发骨架：`plugin.json` + 8 个 skill + OAuth 2.1 + PKCE + DCR |
| 1.0.1 | 重构成 marketplace + 多变体（`smart-tpm-local` / `smart-tpm-test`），`_shared/skills` 物理复制策略落地；删根 `.mcp.json` 与老 `plugin.json` |
| 1.0.2 | `mcpServers` 加 `httpUrl` 字段，兼容 Qwen Code 对 SSE 的误识别 |
| 1.0.3 | 新增 `smart-tpm-prod` 生产变体（初版用 `https://smartquality.bestfunc.com`） |
| 1.0.3-fix | prod URL 切到内网 IP `http://192.168.2.175:8080`（规避自签证书与客户端兼容性问题） |
| 1.0.4 | 新增第 9 个 skill `query-detect-records`（接入 detect_records 模块 3 个 tool） |
| 1.0.5 | 新增第 10 个 skill `about-smart-tpm-mcp`（本百科）；统一各处 scope 描述去掉硬编码 |
| 1.0.6 | 配套后端 v1.5.92：`datasets_*` 6 个工具全部加 `workstation_id` 必填；修 `version_id ≠ dataset_id` 老坑 |
| 1.0.7 | 配套后端 v1.5.93：`files_presign_download` 加 B 模式 `(workstation_id, file_id)`；`query-detect-records` 5 步下载链路、`sample-dataset-extraction` 加下载步骤 |
| 1.0.8 | `sample-dataset-extraction` 加 `datasets_export_manifest`（>20 条走批量分支，1 次拿全量 manifest 替代 N 次 `get_annotations`） |
| 1.0.9 | 配套后端 v1.6.0 新增 31 个 tool：`reproduce-customer-error` + `locate-test-record-algorithm-chain` 接入 `detect_records_trace`（一键 3 层链路）+ `detect_logs_*`（算法节点级 in/out）+ `detect_flow_executions_*`（流程执行实例）+ 元数据反查 + `annotation_history_list` + `audio_ai_req_logs_list` |
| 1.0.10 | 配套后端 v1.6.1：新增 5 个 `marks_*` 写 tool（`mcp:datasets:write` scope）；新增第 11 个 skill `mark-samples-with-ai`（AI 辅助打标 SOP：先 `project_marks_list` 拿字典再用 markCode、`overwrite=false` 安全默认、批量上限 200 条/次）；175 部署侧 tool 总数 58 → 63 |

## 在调研

| 方向 | 现状 | 设想 |
|---|---|---|
| `mcp:detect_flows:write` | 故意不开，防 AI 误改生产流程配置 | 设想加一个"草稿模式"——AI 改的不立刻生效，需 Web 人工 confirm；同时配置 audit log 强化 |
| `mcp:algorithms:write` | 故意不开，防 AI 误改算法参数 | 同上"草稿模式"思路；目前没需求驱动，暂搁置 |
| **更多 task skill** | 当前 6 个 task skill（v1.0.10 后） | 候选方向：算法效果回归对比、批量数据集合并、定时巡检报告生成、AI 自动 cherry-pick 数据集子集做 retrain |
| **官方 marketplace 发布** | 当前仅在 `bestfunc/smart_quality_plugins` 仓库自助安装 | 计划评估是否提交到 Anthropic 官方 marketplace（取决于公司对外开放策略） |
| **prod 域名 + 正规证书** | 当前 prod 走内网 IP HTTP | 设想后续如有公网部署需求，配置正规 CA 签发证书后切回 `https://smartquality.bestfunc.com` |
| **marks 写 tool 风险控制升级** | v1.0.10 首发：`overwrite=false` 安全默认 + 批量上限 200 条 + service 层落审计 | 设想加"AI 打标前必须先抽样 confirm"的 skill SOP，避免 AI 大规模错标无人察觉 |

## 计划中（已立项但未动工）

> 暂无。当前迭代节奏以"按需新增 skill / 修兼容性 bug"为主，没有大规模功能立项。

## 不做（明确不计划的方向）

- **不开放任意 SQL 直查**：所有数据访问必须通过 service 层 + 工位隔离。AI 调用 tool 时只能拿到 service 层 schema 过滤后的字段。
- **不做"AI 全自动运维"**：本插件定位是"AI 编排 + 人工 confirm 写"，不会演进成"AI 自主修改生产配置"。
- **不重复实现 Web 已有的复杂表单**：诸如算法版本对比 UI、检测流程可视化编辑器，仍以 Web 为主。插件只做"读 + 一句话编排"。

## 维护节奏的观察

从 commit 历史看，本仓库的演进模式是：

1. **平台后端先加能力**（Smart TPM 后端注册新 tool）
2. **本仓库追加 skill** 来教 AI 怎么用
3. **bump version → 同步 3 变体 → push**

因此本仓库的迭代频率与 Smart TPM 后端 MCP 模块的迭代频率正相关。如果某个模块在后端长期稳定（如 algorithms 模块），本仓库对应的 skill 也长期不动。

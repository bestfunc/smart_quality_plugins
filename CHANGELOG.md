# CHANGELOG

## v1.0.11

- 配套 web_v2 v1.6.5:后端新增 `datasets_export_files_manifest` —— **专为 0 细标的原始音频数据集**做批量下载,跟 segment 驱动的 `datasets_export_manifest` 互补
  - 走 `version_id` 拿数据集版本全部唯一 fileID(后端按 fileID 去重),响应 `files[].sourceWavUrl` 是 presigned URL,客户端 wget/curl stream 即可,**不切片不要 JSON sidecar**
  - 限额:limit 默认 5000 上限 20000,远高于 export_manifest(2000/5000),给"算法复核/原始语料"这类整集导出留余量
  - scope 复用 `mcp:datasets:read`(老用户零重授权);REST 入口 `POST /api/v2/appapi/v1/dataset/export-files-manifest`(scope `appapi:dataset`)
- `sample-dataset-extraction` SKILL 升级:两路批量 → 三路批量,加判定规则(看 `totalDetailMarkCount` 走 A/B/探查),加流程 B 完整调用样例 + Python wget 脚本
- 后端 tool 总数 63 → 64

## v1.0.10

- 配套 web_v2 v1.6.1:后端加 5 个 `marks_*` 写 tool(scope `mcp:datasets:write`,baseline 已 seed 不需 alembic),AI 现在能直接给样本打**粗标**(`marks_set_rough` 单 channel / `marks_batch_set_rough` 多 channel 一刀)和**细标**(`marks_add_detail` 时间区段 + 类型 / `marks_update_detail` / `marks_delete_detail`),复刻 dataset annotate 页面右键打标的工程师手动流程
- 新增第 11 个 skill `mark-samples-with-ai` —— AI 辅助打标的 SOP,要求先 `project_marks_list` 拿字典再用 markCode、`overwrite=false` 安全默认、批量上限 200 条/次、reproduce 链路靠 `annotation_history_list` 反查
- 写操作全走现有 `DatasetService`,createBy/updateBy 用 `ctx.user_id` 自动填(不是 "MCP"),service 层**自动写 `DetectAnnotationHistory`**,事后可审计「AI 哪次会话改了什么」
- `about-smart-tpm-mcp/reference/04-key-concepts.md` 把 `mcp:datasets:write` 行扩到含 `marks_*` 五个写 tool,提醒老用户重新 OAuth 勾这个 scope 才能用(之前没 tool 用,大概率没勾)
- 175 部署侧后端 tool 总数 58 → 63(本次 v1.0.10 只动 plugin 文档与 skill,backend image 由 web_v2 v1.6.1 提供)

## v1.0.7

- 配套 web_v2 v1.5.93 fix:`files_presign_download` 新增 B 模式 `(workstation_id, file_id)`,后端查工位 file 表把 fileID 解析成 (bucket, path) 再签 URL,返回 url + name + size + mimetype 元数据;A 模式 `(bucket, object_key)` 直传保留
- `query-detect-records`:5 步 `files_presign_download(file_id=...)` 调用改为 `(workstation_id, file_id)`,明示 workstation_id 必填 + 后端解析 fileID
- `sample-dataset-extraction`:加 step 5 下载原始文件(.pcm 音频 / 标注图片),复用 `files_presign_download` B 模式
- 同步配套 backend fix:MinIO presign 的 region 改读 settings.MINIO_REGION(原硬编码 us-east-1),175 部署 .env 加 MINIO_REGION=ap-east-1 对齐 MinIO 服务端配置

## v1.0.6

- 修 `datasets_*` 6 个工具的 skill 文档:全部加 `workstation_id` 必填说明(原 backend bug 已修,server 强制 workstation_id,skill 文档对齐)
  - `sample-dataset-extraction`:rewrite 调用流程,明确 `version_id ≠ dataset_id` 老坑,标 page_size 上限 500
  - `reproduce-customer-error`:`datasets_get_annotations` 调用补 `workstation_id` + `channel_id`
  - `quickstart-smart-tpm-mcp`:加 ⚠️ 段警告 datasets_* / detect_records_* 必传 workstation_id

## v1.0.5

- 新增第 10 个 skill `about-smart-tpm-mcp` —— 插件 / MCP 集成层的自助百科（user-invocable，按角色分流到 reference / scenarios / faq / glossary）
- 统一各处 scope 描述，去掉硬编码"6 个 scope"等过期数字，补 `mcp:detect_records:*` 一致性
- 同步更新 README / 三变体 `plugin.json` description / `marketplace.json` 至 "10 个 skill / 5 个模块"

## v1.0.4

- 新增第 9 个 skill `query-detect-records`，接入生产 175 的 detect_records 模块（3 个 tool：`_list` / `_get` / `_ping`，工位由 `detect_stations_list` 列出，大文件走 `files_presign_download`）

## v1.0.3

- 新增 `smart-tpm-prod` 生产变体；prod URL 切到内网 IP `http://192.168.2.175:8080`（规避自签证书与客户端兼容性问题，原域名 `https://smartquality.bestfunc.com` 暂不用）

## v1.0.2

- `mcpServers` 三变体均加 `httpUrl` 字段，与 `url` 同值，兼容 Qwen Code 对 SSE transport 的误识别

## v1.0.1

- 重构为 marketplace + 多变体结构：根目录 `.claude-plugin/marketplace.json` 声明 3 个变体（`smart-tpm-local` / `smart-tpm-test` / `smart-tpm-prod`，1.0.1 时只有前两个）
- `_shared/skills/` 物理复制到 3 个变体目录（Windows git symlink 不稳，故采用物理拷贝）
- 删除根 `.mcp.json` 与老 `plugin.json`

## v1.0.0

首发：

- `plugin.json` 声明 MCP server `smart-tpm`（HTTP transport）
- 8 个 Skill（1 Onboarding + 3 Reference + 4 Task），覆盖 datasets / detect_flows / test_tasks / algorithms 4 个模块
- 鉴权走 OAuth 2.1 + PKCE + DCR，配合 Smart TPM Web v1.5.44+

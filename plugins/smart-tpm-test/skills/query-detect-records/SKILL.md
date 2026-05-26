---
name: query-detect-records
description: 用户拿一条客户报错信息(产品编号 / 时间 / 工位) 或想抽样查质量,想用 MCP 查生产线 detect_channel 检测记录时使用。本 skill 教模型用 detect_records_list/get 配合 files 模块拉原始数据,且强调工位分库注意点。
allowed-tools: detect_records_ping, detect_records_list, detect_records_get, detect_stations_list, files_presign_download, detect_flows_get
---

# query-detect-records

## 何时用

- 客户工程师反馈"X 产品在 Y 工位 Z 时间 NG/OK 错了",想复现
- 数据团队抽样:"今天迈腾工位 NG 前 20 条"
- 算法回归对比:"对比 detect_channel 历史 aiResult vs 新算法跑出来的"
- 想拉某条检测的原始音频/图片做主观 check

## 必填的 workstation_id

**所有 detect_records 工具第一个参数都是 `workstation_id`,必填。** 数据按工位**物理分库**,每个 workstation 自己一个 MySQL 实例,主库只存 workstation 元数据(连接凭证)。

如果不知道 workstation_id,先调 `detect_stations_list()` 拿全集再选;或问用户"这个客户在哪个工位"。

## 用法流程

### 场景 A: 复现客户报错

1. 拿到客户报的"工位 / 产品编号 / 时间 / 报错描述"
2. `detect_records_list(workstation_id, productUniqueCode='xxx', dateFrom='YYYY-MM-DD', dateTo='YYYY-MM-DD')` — 通常带 `result='NG'` 缩窄
3. 看返回的三级嵌套:`items[*].detectPoints[*].stages[*]`,找到客户描述的那条,记下 `channelID`
4. `detect_records_get(workstation_id, channel_id)` 拿完整 row:
   - 三个 `result` 字段(userResult/aiResult/plcResult)的差异
   - `fileID` 是 MinIO 原始音频/图片对象
   - `roughProjectMarkJson` / `detailedProjectMarkJson` 是标注详情
   - `detectStageID` / `detectPointID` 串到检测流程哪一步
5. 如要听原始音频:`files_presign_download(workstation_id, file_id=<fileID>)` 拿带签名的下载 URL。**workstation_id 必传**,后端查工位 file 表把 fileID 解析成 (bucket, path) 再签,返回 url + name + size + mimetype;不传只签 (bucket, object_key) 直传模式
6. 如要看检测流程定义:用 record 里的 detectID 串 `detect_flows_get(...)`(注意:detect_flows 在主库,不分工位)

### 场景 B: 抽样质检

1. `detect_records_list(workstation_id, result='NG', page=1, page_size=20, sortOrder='desc')` 拿最新 NG 20 条
2. 视情况按 `detectStationID` / `detectPointID` 进一步过滤
3. 抽几条 `detect_records_get` 详查

### 场景 C: 验证工位连通(troubleshooting)

工具调失败时先用 `detect_records_ping(workstation_id)`:
- 返回 `{ok: true, mysql_version: '8.x.x'}` → 工位路由正常,问题在业务 SQL 或数据
- 抛 "workstation not found" → 工位 ID 错 / 工位被删 / status 不是 active
- 抛 "decrypt db_password failed" → 工位凭证加密损坏,联系运维

## 限制

- **只读**。首期不提供 write 工具(写 detect_channel 会扰动生产线数据)
- **不跨工位聚合**(如"全工位今日 NG 率")。要这种统计请客户调单工位多次然后客户端拼。
- 高级 `filters` JSON 参数本期未开,只支持 list 入参里列出的明确字段
- 大数据集查询请带 `dateFrom`/`dateTo` 范围,否则单工位过去几个月数据会很慢

## 常见错误

- "tool not found: detect_records_list" — 客户端拉的 plugin 版本旧,跑 `/plugin marketplace update` 后重启
- 401 `WWW-Authenticate: Bearer` — token 没 `mcp:detect_records:read` scope,重新走 OAuth 同意页勾选这个 scope
- 三级嵌套 items 是空但 total > 0 — 检查 page 是否超过 total/page_size

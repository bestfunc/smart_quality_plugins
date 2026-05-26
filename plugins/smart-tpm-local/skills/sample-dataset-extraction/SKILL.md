---
name: sample-dataset-extraction
description: 用户想从某个数据集抽样本看标注（如"抽 NG 样本前 20 条"、"看下数据集 X 里 label=瑕疵 的最近样本"）时使用。
allowed-tools: datasets_list, datasets_get, datasets_query_samples, datasets_get_annotations
---

# 抽取数据集样本

## ⚠️ workstation_id 必填

datasets_* 全部 6 个工具(list/get/query_samples/get_annotations/create_draft/import_items)都**必须**传 `workstation_id` —— 数据集、版本绑定、样本通道全在工位库,主库返空。

如果用户没给工位:
- 当前 IDE 上下文里如果有 `X-Workstation-Id`(常见于 SPA 当前工位),直接复用
- 否则**问一句"哪个工位的数据集?"** 别瞎传

## 流程

1. **定位数据集**:如用户已给数据集名/ID,跳到 2;否则先 `datasets_list(workstation_id, q=...)` + 关键字过滤,让用户确认
2. **看数据集结构**:`datasets_get(workstation_id, id)` 拿到 `versions` 数组 + 元数据 —— **不报告完整 schema 给用户**,只挑用户关心的字段(如版本号 + dataCount + label 候选列表)
3. **抽样本**:`datasets_query_samples(workstation_id, version_id, filters, page_size)`
   - `version_id` 必填(从 step 2 的 `versions[*].id` 拿,**不是** dataset.id)
   - `filters` 是 JSON 字符串(如 `'[{"field":"productUniqueCode","operator":"contains","value":"NG"}]'`),不传则全量
   - `page_size` 默认 50,最大 500
4. **看标注详情**(可选):对挑出的样本,对每个 `channel_id` 调 `datasets_get_annotations(workstation_id, channel_id)`
5. **下载原始文件**(可选):样本 channel 里有 `fileID`(音频 .pcm / 图片)、细标 mark 里也有 `fileID`,调 `files_presign_download(workstation_id, file_id)` 拿 15 分钟内有效的下载 URL,客户端直接 curl/wget,不走后端带宽

## 输出格式

按下面格式给用户简表:

| 数据编号 | label | markCode | 标注时间 | 标注人 |
|---|---|---|---|---|

如用户要详情,再展开 ROI(细标)。

## 注意事项

- 全部 datasets_* 工具都要 `workstation_id`,漏传 server 直接返 schema validation 错误
- `version_id` 跟 `dataset_id` 是两个东西:`datasets_get` 拿 dataset 详情,返回里的 `versions[*].id` 才是 query_samples 要的 version_id
- `datasets_query_samples` 的 `page_size` 默认 50、最大 500
- **不要循环调** `datasets_get_annotations`,N 次 sample = N 次 API;超过 50 个样本建议先让用户缩窄筛选
- 跨组织/跨工位样本你**看不到** —— ws_db 物理隔离 + service 层 SQL WHERE 双重隔离

## 反例

❌ 用户问"看下数据集 X" → 直接 `datasets_query_samples(version_id=...)` 不传 workstation_id → 报缺参 / 返空。
❌ 把 `dataset_id` 当 `version_id` 传给 query_samples → 总是 total:0(子查询 `DetectChannelVersion.versionID` 不匹配)。
✅ 先 `datasets_get(workstation_id, id)` 拿版本数组,告诉用户"V1.0 有 6294 条,要按啥条件筛?",再 query_samples 传对应 version.id。

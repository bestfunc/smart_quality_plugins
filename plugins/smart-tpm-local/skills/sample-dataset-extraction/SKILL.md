---
name: sample-dataset-extraction
description: 用户想从某个数据集抽样本看标注（如"抽 NG 样本前 20 条"、"看下数据集 X 里 label=瑕疵 的最近样本"）时使用。
allowed-tools: datasets_list, datasets_get, datasets_query_samples, datasets_get_annotations
---

# 抽取数据集样本

## 流程

1. **定位数据集**：如用户已给数据集名/ID，跳到 2；否则先 `datasets_list` + 关键字过滤，让用户确认
2. **看数据集结构**：`datasets_get(dataset_id)` 拿到 `annotationSchema` —— **不报告完整 schema 给用户**，只挑用户关心的字段（如 label 候选列表）
3. **抽样本**：`datasets_query_samples(dataset_id, filters, limit)`，常见 filters：
   - `label` = 某个粗标值（如 `NG / OK`）
   - `markCode` = `fine_done / coarse_done`
   - `startTime / endTime` = 时间段
   - `keyword` = 数据编号片段
4. **看标注详情**（可选）：对挑出的样本，对每个 sample_id 调 `datasets_get_annotations`

## 输出格式

按下面格式给用户简表：

| 数据编号 | label | markCode | 标注时间 | 标注人 |
|---|---|---|---|---|

如用户要详情，再展开 ROI（细标）。

## 注意事项

- `datasets_query_samples` 默认 `limit=20`，最大支持的 limit 见后端报错（一般 ≤ 200）
- **不要循环调** `datasets_get_annotations`，N 次 sample = N 次 API；超过 50 个样本建议先让用户缩窄筛选
- 跨组织样本你**看不到**——service 层 SQL WHERE 已隔离

## 反例

❌ 用户问"看下数据集 X" → 直接 `datasets_query_samples(limit=200)` 灌一屏。
✅ 先 `datasets_get` 拿 sample 总数，告诉用户"X 数据集有 12,345 条，要按啥条件筛？"

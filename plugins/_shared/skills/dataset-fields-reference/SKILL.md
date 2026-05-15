---
name: dataset-fields-reference
description: 用户问数据集相关字段含义（粗标 / 细标 schema、sampleRatio、markCode 锁定规则等）时使用。纯文档 Skill,无工具调用。
---

# 数据集字段速查

## 数据集主表关键字段

| 字段 | 类型 | 含义 |
|---|---|---|
| `datasetID` / `id` | UUID String(32) | 主键 |
| `datasetName` | string | 数据集名 |
| `companyId` | UUID String(32) | 组织隔离键，**所有查询必带** |
| `projectId` | UUID String(32) | 所属项目 |
| `productID` | UUID String(32) | 所属产品 |
| `annotationSchema` | JSON | 标注 schema(粗标 / 细标的 label 候选列表、必填字段) |
| `sampleRatio` | float (0~1) | 默认抽样比例,被测试任务复制为初始值 |

## 标注 schema 的两层

### 粗标（coarse annotation）

- **样本级**标注（整张图 / 整条记录）
- 主键 `label`：业务定义的分类标签（如 `OK / NG / unknown`）
- 字段：`label`, `confidence`(可选), `markedBy`, `markedAt`

### 细标（fine annotation）

- **ROI 级**标注（一张图里框出多个区域）
- 主键 `roiId` + `label`
- 字段：`roiId`, `bbox`(x/y/w/h), `label`, `polygon`(可选), `markedBy`, `markedAt`

## markCode 锁定规则

`markCode` 字段（在 sample 上）= 标注完成度状态机：
- `unmarked`：未标注
- `coarse_done`：粗标完成
- `fine_done`：细标完成（自动包含粗标完成）
- `locked`：人工锁定，**不再允许任何修改**

**`locked` 是终态**。MCP 写工具 `datasets_import_items` 不会反向解锁。

## sampleRatio 含义

抽样比例，0~1 之间的浮点数。语义：从数据集里**按这个比例**抽样进入测试任务。

- `0.3` = 抽 30%
- 数据集层 sampleRatio 是默认值；测试任务里可覆盖
- 配合 `markCode` 过滤：常见组合是"`markCode=fine_done` + sampleRatio=0.5"

## 近期 ORM 变更（commit 提示）

如发现某字段实际行为与本文档不符，**首先查 git log**：

```bash
git log -p -- smart_tpm_edge_api_v2/app/models/dataset.py | head -100
```

本文档随主项目 v1.5.44 同步；以代码为准。

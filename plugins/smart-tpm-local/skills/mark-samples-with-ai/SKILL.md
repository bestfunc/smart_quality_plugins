---
name: mark-samples-with-ai
description: 用户让 AI 给样本/通道打粗标(整段分类)或细标(时间区段)时使用,如「把这批 200 个 OK 的样本统一标 OK」「给这个 channel 在 1.5s-2.0s 加 NG002」。写操作,需要 mcp:datasets:write 授权。
allowed-tools: project_marks_list, datasets_query_samples, datasets_get_annotations, rough_marks_list, detail_marks_list, annotation_history_list, marks_set_rough, marks_batch_set_rough, marks_add_detail, marks_update_detail, marks_delete_detail
---

# AI 辅助打标

> **写操作**:这 skill 涉及修改样本标注,需要用户授权时勾上 `mcp:datasets:write`。
> 所有写操作 service 层自动写 `DetectAnnotationHistory` 审计表,事后可通过 `annotation_history_list` 反查。

## 三种粒度

| 粒度 | tool | 场景 |
|---|---|---|
| **粗标**(单 channel 整段分类) | `marks_set_rough` | 「把 channel X 标成 OK」 |
| **粗标批量**(多 channel 一刀) | `marks_batch_set_rough` | 「这批 200 个都是 OK」 |
| **细标**(channel 内时间区段) | `marks_add_detail` | 「channel X 的 1.5s-2.0s 标 NG002」 |
| **细标改/删** | `marks_update_detail` / `marks_delete_detail` | 修标错的细标 |

## 标准流程

### 1. 先拿可用标签字典(别瞎编 markCode)

```
project_marks_list(project_id="<项目 ID>")
→ 得到该项目预定义的 [{markCode, markName, markColer, sampleType}] 列表
```

打标时 `mark_code` / `mark_name` 必须用字典里有的,**不要硬编码** `"NG"` / `"OK"`。

### 2. 找到要打的 channel(粗标场景)

按需选一条路径拿 channel_ids:

- 「这个数据集里所有未标的样本」→ `datasets_query_samples(version_id, has_rough_mark=false)`
- 「这个工位最近 24h 的 NG channel」→ `detect_records_list(...)` 或工位 channel 直接搜
- 用户已经给你 channel_ids → 直接用

### 3. 打标

**批量粗标**(强烈推荐用于多 channel):
```
marks_batch_set_rough(
  workstation_id=...,
  channel_ids=[...],
  mark_code="OK",
  mark_name="OK",
  mark_color="green",
  overwrite=false,   # 默认 false:已标但不同的进 conflicts 不动,只标未标的
)
→ 返回 {updatedCount, skippedCount, conflictChannelIDs, conflictDetails}
```

**单 channel 粗标**:
```
marks_set_rough(workstation_id=..., channel_id=..., mark_code="NG001", mark_name="NG001", mark_color="red")
```

**细标**(给 channel 加时间区段标记):
```
marks_add_detail(
  workstation_id=...,
  channel_id="...",
  items=[
    {"mark_code": "NG002", "mark_name": "NG002", "mark_color": "red", "start": 1.5, "end": 2.0},
    {"mark_code": "NG002", "mark_name": "NG002", "mark_color": "red", "start": 3.2, "end": 3.5},
  ],
)
```

### 4. 报告 + 留痕

打完汇报:
- 实际改了几条(`updatedCount`)、跳过几条(`skippedCount`)、冲突几条(`conflictChannelIDs`)
- 如有 conflict,**默认行为是不动它们**;若用户明确说「全覆盖」再带 `overwrite=true` 重跑
- 提示用户:`annotation_history_list(channel_id=...)` 可反查这次改了什么

## ⚠️ 安全护栏 / 不要做的事

- **不要**自行决定 `overwrite=true` —— 必须用户明确说「覆盖」「强标」「不管原来的」才传 true
- **不要**对不存在的 markCode 打标 —— 先 `project_marks_list` 拿字典,再用字典里的 code
- **不要**对粗标用空 `mark_code` 当万能值清除 —— 空串是合法的"清除粗标"语义,确认用户真要清再做
- **不要**跨工位批量 —— `workstation_id` 一次只能传一个,跨工位让用户分多次调
- **不要**单次 `channel_ids` 超过 500 条 —— 后端 `maxItems: 1000`,但分批 200 条一次更稳

## 反查 / 撤销

误操作或要审计:
```
annotation_history_list(workstation_id=..., channel_id=..., date_from="<今天>", operation_type="update")
→ 看 createBy / operationContent,createBy 是你(被授权 AI 调用方),不是 "MCP"
```

撤销细标:`marks_delete_detail(workstation_id, mark_id)`(软删,history 留底)
撤销粗标:重新 `marks_set_rough(..., mark_code="")` 清掉,或还原成原 mark_code

## 注意事项

- 标 NG/NG001/NG002 时的颜色用红色,OK 用绿色,跟项目字典对齐;具体颜色 ID(`red` / `green` / `blue` / ...)看 `project_marks_list` 返回的 `markColer` 字段
- 时间单位都是**秒**(float),不是毫秒
- `markIndex` 后端自动递增,不要传
- 工位库分库,`workstation_id` 必填,后端强制

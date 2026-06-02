# 04 · 检测引擎 · dataContent 协议 · 双视角聚合

> 事实来源：`docs/detect_execute_api.md`、`docs/detect_execute_response.md`、`v2/detect/`。对接方/排障必读。

## 定义：检测引擎做什么

检测引擎（DetectEngine）是**业务编排层**：把外部系统推来的一批检测数据，拆成多个 stage，每个 stage 调一次算法 DAG 子流程，最后从两个视角聚合成产品级判定。

对外接口：`POST /api/v2/detect/execute`，`Content-Type: application/json`（推荐），鉴权 `X-API-Key`（需 `detect_engine:execute` 权限）。

## 细节：请求参数

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `dataContent` | object | 是 | — | 五张表数据（见下） |
| `flowNumber` | string | 是 | — | 检测流程编号，如 `"TP-0009"` |
| `versionCode` | string | 否 | 最新启用版本 | 流程版本号；**生产建议显式指定**避免版本漂移 |
| `mode` | string | 否 | `"0"` | `"0"`=实际 / `"1"`=全 OK / `"2"`=全 NG（1/2 仅调试） |
| `device_type` / `device_id` | string | 否 | `""` | 设备标签，透传到算法层 |
| `disabledRules` | object | 否 | — | **测试任务**禁用规则（v0.13.7+），传入则**完全覆盖** `flowData.stageAssignments[*].disabled` |

### dataContent 五张表

```
detect（检测单，至少 1 条，取 detect[0].id）
  └─ detect_point（检测点位）        ← detect_channel.detectPointID 引用
       └─ detect_channel（检测通道，工位+阶段） ← detect_channel.fileID 引用 file.id
            └─ file（音频文件）      ← path 指向 MinIO 对象
  detect_station_stage（工位阶段，当前预留）
```

约束：`isDelete="1"` 的记录自动过滤；`detectPointID` 必须能在 `detect_point` 找到；`file.path` 必须是有效 MinIO 路径；`hallValues` 会注入算法 `external.hallValues`（**驼峰键**，节点通过 `line_data_json_key="hallValues"` 读取）。

### disabledRules 结构

```json
{ "points": ["pointID_A"],
  "stages": [{"pointID":"pointID_A","stageIndex":1}, {"pointID":"pointID_B","stageIndex":3}] }
```

- 算法侧只看 `stages`，`points` 仅冗余索引
- **不传** = 沿用 flowData 自带 `disabled`（生产路径）
- **传空 `{"points":[],"stages":[]}`** = 完全覆盖 = 所有 stage 启用

## 示例：响应结构与双视角聚合

接口同时返回两种聚合视图，前端按需选用：

| 视图 | 聚合方式 | 适用场景 |
|---|---|---|
| `point_first` | 先按**点位**聚合 → 产品判定 | 关注每个检测点位的综合结果 |
| `stage_first` | 先按**阶段/工位**聚合 → 产品判定 | 关注每个工序阶段的通过率 |

```jsonc
{ "code": 200, "message": "检测完成", "data": {
    "point_first": { "productResult": "positive",   // positive=合格 / negative=不合格
      "pointResults": [ { "pointKey":"point-001", "result":"positive",
        "matchCount":1, "sampleCount":1, "stageResults":[ {...} ] } ] },
    "stage_first": { "productResult": "positive", "stageResults":[ {...} ] },
    "stageDetails": [ { "stageKey":"point-001__stage1", "pointID":"point-001",
        "channelID":"ch-001", "flowNumber":"FLW-xxx", "versionCode":"V1.0",
        "duration":1523.45, "status":"success", "source":"executed",
        "errorMsg":null, "resultRaw":{...} } ],
    "warnings": [ "..." ]   // 仅异常时出现
} }
```

### 关键字段

| 字段 | 取值 / 说明 |
|---|---|
| `productResult` | `positive`=合格 / `negative`=不合格 |
| `stageDetails[].status` | `success` / `fail` / `algorithm_error` / `cached` / `skipped` |
| `stageDetails[].source` | `executed`（本次执行）/ `cached`（复用缓存）/ `skipped`（无文件跳过） |
| `stageKey` | 唯一标识 `pointID__stageN` |
| `warnings` | 诊断数组，正常流程不出现；`matchCount=0` 时优先看这里 |

## 异常可追溯（detect_log 三表）

执行日志异步写入 MySQL **三张审计表**（detect_log_* 系列），是**只写不改不删**的审计表，禁止 UPDATE / DELETE 历史记录。

> **红线**（CLAUDE.md / git log）：
> - `_detect_soft_failure` 必须同时查嵌套 `ai_result.code`，否则算法异常会被吞为 success
> - 节点新增异常路径必须先写 `ctx.node_logs` 的 fail 条目再 raise，保证 detect_log_node 有 partial 记录
> 详见 [faq/02-ops-questions.md](../faq/02-ops-questions.md) 与 `docs/detect_log_tables_guide.md`。

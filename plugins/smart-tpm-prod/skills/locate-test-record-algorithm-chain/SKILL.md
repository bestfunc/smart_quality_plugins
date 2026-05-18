---
name: locate-test-record-algorithm-chain
description: 用户给一个 test_record id 或编号,想追溯它走了哪条 detect_flow、哪个算法流、用了哪个算法 + 阈值时使用。
allowed-tools: test_records_get, detect_flows_get, detect_flows_list_nodes, algorithm_flows_get, algorithms_get
---

# 定位测试记录算法链路

## 链路结构

```
test_record (testRecordId,含 detectFlowId + detectFlowVersionId)
 → detect_flow (按 record 里锁定的版本)
   → detect_flow_node[] (流程节点定义)
     → algorithm_flow (algorithm 节点绑的算法流)
       → algorithm (算法流里的具体算法)
```

> 注: detect_flow_executions tool(每节点实际输出+tags)首期未提供。要看一次执行的逐节点实际输出,转 Web UI: `/test-tasks/<taskId>/records/<recordId>`。

## 流程

1. **拿 record**: `test_records_get(record_id)` → 得到 `taskId`, `detectFlowId`, `detectFlowVersionId`(record 里锁定的执行时流程版本)
2. **拿流程定义**: `detect_flows_get(flow_id, version_id)` → 得到节点列表 + 每个节点的 `currentNodeAttr`
   - 如需扁平节点列表跨版本对比,补一刀 `detect_flows_list_nodes(flow_id)`
3. **逐节点追算法**: 对每个 algorithm 类型节点,取 `currentNodeAttr.algorithmFlowVersionId`,调 `algorithm_flows_get(flow_id, version_id)`
4. **拿具体算法 + 参数**: 算法流里每个算法节点 → `algorithms_get(algorithm_id)`,重点报告参数(阈值 / 模型路径)

## 输出格式

```
test_record #<编号>
└─ detect_flow_execution: <executionId>
   └─ detect_flow: <flowName> (v<version>)
      ├─ Node 1 (algorithm) → algorithm_flow: <name> v<version>
      │    └─ Algorithm: <name>, 阈值: confidence>0.85
      ├─ Node 2 (merge) → tags: ["温度异常"], show_model_result=true
      └─ Node 3 (output) → label: NG
```

## 注意事项

- 一条 test_record 可能对应**多个版本**的 detect_flow（按执行时刻锁定）—— 必须用 `detectFlowVersionId` 不是当前版本
- 中途如发现算法/流程已 soft-delete，正常报告"该算法已下线但执行历史保留"
- 不要替用户做"应该这样改阈值"的建议 —— 这是只读追溯，结论由工程师决定

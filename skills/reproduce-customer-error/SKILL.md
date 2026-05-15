---
name: reproduce-customer-error
description: 用户(工程师)拿到客户报错的描述 + 一条 test_record id,想梳理上下文给出"可能原因 + 复核步骤"清单时使用。纯只读追溯,不做修改。
allowed-tools: test_records_get, test_tasks_get, detect_flows_get, detect_flows_list_nodes, algorithm_flows_get, algorithms_get, datasets_get, datasets_get_annotations
---

# 复现客户报错

## 输入

- 一条 test_record id（或编号）
- 客户的错误描述（如"明明是 OK 的样本被判成 NG"）

## 流程

1. **拿 record**: `test_records_get(record_id)`,记录关键字段:
   - 最终 label / score
   - taskId / detectFlowId / detectFlowVersionId(record 里锁定的执行时版本)
   - sampleId(指向数据集里的原样本)
   - createdAt
2. **看任务上下文**: `test_tasks_get(task_id)`,看:
   - sampleRatio
   - coarseFilters / fineFilters
   - 创建时间 + 创建人
3. **看流程定义**: `detect_flows_get(flow_id, version_id)` —— 用 record 里锁定的 `detectFlowVersionId`,不是当前版本;如需扁平节点列表补一刀 `detect_flows_list_nodes(flow_id)`
4. **看样本标注**: `datasets_get_annotations(sample_id)` —— 比对**实际标注**与 record 判定结果的差异
5. **看算法**: 对节点中 model 类的算法,`algorithms_get(algorithm_id)` 看参数 + 阈值

> 注: detect_flow_executions tool(每节点实际输出+tags)首期未提供。要看一次执行的逐节点 raw 输出,引导工程师打开 Web UI: `/test-tasks/<taskId>/records/<recordId>`。

## 输出格式（给工程师）

```
## 案件概览
- record: <编号>, 实际判定: <label>, 客户期望: <label>
- 任务: <taskName> (id <taskId>, 创建于 <date>)
- 流程版本: <flowName> v<version>(注意是执行时锁定的版本)

## 可能原因(优先级 ↓)
1. <原因 1：基于哪个证据>
2. <原因 2：基于哪个证据>

## 复核步骤
- [ ] 步骤 1：登录 Web,打开 /test-tasks/<taskId>/records/<recordId>,核对节点 2 的输出
- [ ] 步骤 2:查算法 <X> 的阈值是否近期调过(`git log -p -- ...`)
- [ ] 步骤 3:......
```

## 注意事项

- **不要**做修改性建议(如"把阈值改成 0.7")—— 这是诊断,不是处方
- 报告 **哪些证据来自哪个 tool**,让工程师能自己复核
- 如果中途发现该 record 跨组织(理论上看不到),告诉用户"权限不足,请联系超管"
- 客户描述里的口语化词汇(如"误检""漏检")要在报告里对应到 confusion matrix 的精确术语

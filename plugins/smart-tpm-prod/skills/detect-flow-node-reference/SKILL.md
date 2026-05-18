---
name: detect-flow-node-reference
description: 用户问检测流程节点的字段含义（合并节点的 show_model_result / skip_model_result / add_tag 与 current_node_attr 关系）时使用。纯文档 Skill,无工具调用。
---

# 检测流程节点速查

## 节点（detect_flow_node）关键字段

| 字段 | 含义 |
|---|---|
| `nodeId` | 节点主键 |
| `nodeType` | `algorithm / merge / branch / output` |
| `algorithmFlowId` | 当节点类型 = algorithm 时，绑定的算法流 |
| `currentNodeAttr` | JSON，节点级配置（见下文） |
| `inputs` / `outputs` | 上下游节点的连接关系 |

## 合并节点（merge node）三个关键开关

合并节点用于把多个上游算法的输出合并成最终的检测结论。三个核心 flag（来自 commit `02f0613`）：

### `show_model_result`（布尔）

- `true`：在 Web UI 上**展示**该上游模型的结果作为参考
- `false`：不展示（一般用于辅助类模型，不希望操作员看到）

### `skip_model_result`（布尔）

- `true`：合并时**忽略**该上游模型结果（不参与最终判定）
- `false`：参与判定

> 注意：`show_model_result=true` + `skip_model_result=true` 是合法组合 = 展示但不参与判定（仅 UI 显示，不影响结果）。

### `add_tag`（字符串或 null）

- 给该上游模型贡献的结果**打 tag**
- 出现在 detect_flow_execution 的 tags 列表里
- 常用于在多模型组合的合并结果上做事后分析（如 tag="温度异常"）

## current_node_attr 的角色

`currentNodeAttr` 是节点自身配置（JSON），**与上面三个 flag 的关系**：

- 合并节点的 `show_model_result / skip_model_result / add_tag` 是**每个 input 一组**（保存在 `currentNodeAttr.inputs[]` 里）
- 节点级的全局开关（如该合并节点是否启用）在 `currentNodeAttr.enabled`
- algorithm 节点的算法版本绑定在 `currentNodeAttr.algorithmFlowVersionId`

## 想看实际数据？

试 MCP tool：

````
detect_flows_get(flow_id="...")             # 完整流程 + 全部节点
detect_flows_list_nodes(flow_id="...")      # 列指定流程下的所有版本(含节点 flowData)

# 注: detect_flow_executions tool 首期未提供
# 想看一次执行的实际节点输出 + tags,转 Web UI:
#   /test-tasks/<taskId>/records/<recordId>
````

---
name: create-test-task-from-template
description: 用户想仿造某个历史测试任务(改部分参数如 sampleRatio / 粗标筛选 / 细标筛选)新建一份草稿时使用。这是 MCP 插件唯一的写操作场景。
allowed-tools: test_tasks_list, test_tasks_get, test_tasks_create_from_template
---

# 基于模板新建测试任务

## 前置确认

**写操作需要 `mcp:test_tasks:write` scope**。授权页未勾选时 `test_tasks_create_from_template` 会被 dispatcher 拒绝（错误码 -32002 SCOPE_DENIED）。如果调用失败,告诉用户去 Smart TPM Web → 个人设置 → 已连接的应用 → 撤销重授权时勾上 write scope。

## 流程

1. **定位模板**：如用户给了模板 id 直接跳 2；否则 `test_tasks_list(filters)` 列出候选,关键字段：`taskName / createdAt / sampleRatio`
2. **看模板细节**：`test_tasks_get(task_id)` → 得到 `detectFlowId / sampleRatio / coarseFilters / fineFilters / dataSetIds`
3. **跟用户对齐改什么**：报告"基于模板 X，默认抽样 50%、过滤 label=NG。要改哪些参数？"等用户回应
4. **调创建**：`test_tasks_create_from_template(template_task_id, overrides={...})` 创建**草稿**(status=draft)
5. **报告结果**：返回新建的 task_id 和 Web 链接「/test-tasks/edit/{id}」让用户去补完后启动

## overrides 字段（常见）

| 字段 | 类型 | 说明 |
|---|---|---|
| `taskName` | string | 新任务名(不建议跟模板同名) |
| `sampleRatio` | float | 抽样比例覆盖 |
| `coarseFilters` | dict | 粗标筛选(如 `{"label": "NG"}`) |
| `fineFilters` | dict | 细标筛选 |
| `datasetIds` | list[str] | 切换数据集(慎用) |

## 注意事项

- 创建的任务**是草稿,不会自动启动** —— 草稿状态在 Web 上可以继续编辑
- 不要替用户**改 detectFlowId**(算法链路换了等于另一个任务,语义偏差太大)
- 失败时打印**完整错误**给用户(常见原因：scope 缺失、模板已 soft-delete、用户无该工位写权限)

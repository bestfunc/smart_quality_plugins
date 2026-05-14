---
name: quickstart-smart-tpm-mcp
description: 用户第一次安装 Smart TPM MCP 插件、问"能干啥""怎么用""有哪些功能"时使用。本 Skill 给出 4 个模块总览 + 第一条对话示例。
allowed-tools: datasets_list, detect_flows_list, test_tasks_list, algorithms_list
---

# 快速上手 Smart TPM MCP

Smart TPM MCP 插件把工厂检测平台的能力暴露给 AI，让工程师在 Claude Code 里用自然语言完成原本要跨多个页面才能做的查询和初稿配置。

## 4 个模块能干啥

- **datasets（数据集）**：列出当前组织可见的数据集；按标注/时间段/数据编号抽样本；查看标注（粗标 / 细标）；创建数据集草稿；按数据编号导入样本
- **detect_flows（检测流程）**：列出当前工位的检测流程；看流程详情（节点 / 参数 / 阈值）；查检测流程的执行记录
- **test_tasks（测试任务）**：列出/查看测试任务和测试记录；**基于模板创建新测试任务（草稿）**——这是本插件唯一的写操作
- **algorithms（算法）**：列出算法库、算法流、参数模板（首期只读）

## 第一条对话示例

````
> 帮我列下产品"装配线A"上启用的检测流程
[AI 调 detect_flows_list ...]

> 抽一下数据集 "缺陷-NG-2026Q1" 里 label=NG 的样本前 20 条
[AI 调 datasets_query_samples ...]

> 仿造测试任务 #1024 模板新建一份(改 sampleRatio 为 0.3)
[AI 调 test_tasks_get + test_tasks_create_from_template ...]
````

## 隔离与权限

- 你只能看到自己**组织 + 工位**下的数据（service 层强制隔离）
- 写操作必须在 OAuth 授权页**显式勾选**（`mcp:*:write` scope）
- 所有调用都会落 `mcp_audit_log` 表，超管可在 Web → 系统设置 → MCP 审计 看到

## 想做更具体的事？

- 抽样本看标注 → 试 Skill **"抽取数据集样本"**
- 给一条 test_record 追算法链路 → 试 Skill **"定位测试记录算法链路"**
- 仿造一份测试任务 → 试 Skill **"基于模板新建测试任务"**
- 客户报错要复盘 → 试 Skill **"复现客户报错"**

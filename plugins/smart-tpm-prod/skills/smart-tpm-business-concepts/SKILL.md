---
name: smart-tpm-business-concepts
description: 用户问 Smart TPM 的业务概念、术语关系（项目/产品/工位/检测点/阶段/检测流程/测试任务/数据集 是什么、彼此关系）时使用。纯文档 Skill,无工具调用。
---

# Smart TPM 业务概念速查

## 实体关系总览

```
组织 (company)
 └─ 项目 (project)
     └─ 产品 (product)
         └─ 工位 (workstation)
             ├─ 检测点 (detect_point)            ← 物理位置
             │   └─ 阶段 (detect_stage)         ← 检测的时序步骤
             │       └─ 检测流程 (detect_flow)  ← 实际跑的算法工作流
             │           └─ 节点 (detect_flow_node)
             │
             └─ 测试任务 (test_task)            ← 跑流程产生的批次
                 └─ 测试记录 (test_record)     ← 单条样本的执行结果
                     └─ detect_flow_execution  ← 流程在该记录上的执行实例
```

## 关键术语

- **组织（company）**：租户单位，**所有数据按 `companyId` 严格隔离**
- **工位（workstation）**：物理产线工位；用户的可见性按工位组装（参考 `WorkstationService.get_user_workstations`）
- **检测点 (detect_point) → 阶段 (detect_stage) → 检测流程 (detect_flow)**：三层时序结构。一个阶段绑一个检测流程
- **检测流程（detect_flow）**：算法工作流的实际"图"。版本化（`detect_flow_version`），生产跑的是某个 published 版本
- **算法（algorithm）**：单个算法的注册条目；**算法流（algorithm_flow）** = 多算法编排的工作流（与 detect_flow 解耦）
- **算法参数（algorithm_param）**：算法的参数模板，可以被多个 detect_flow 引用
- **数据集（dataset）**：标注样本集合。**粗标**（coarse）= 整张图/记录级；**细标**（fine）= ROI 级
- **样本（sample / dataset_item）**：数据集里的一条数据（一个图 + 标注）
- **测试任务（test_task）**：一次批量跑 detect_flow 的批次。`sampleRatio` 决定抽多少比例的样本进来
- **测试记录（test_record）**：测试任务执行后的单条产物，对应 detect_flow_execution

## 与 ApiKey 体系的区别

- **本 MCP 插件**（OAuth）= 你**本人**的 AI 客户端，绑定 user + org
- **ApiKey + /auth/verify-key** = 服务到服务（如 `smart-tpm-mcp-server` 中转层）的鉴权，不绑用户
- 两者走同一个 `/api/v2/mcp` 端点，handler 不区分来源

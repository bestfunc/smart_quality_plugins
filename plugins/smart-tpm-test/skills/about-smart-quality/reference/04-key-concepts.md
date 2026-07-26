# 核心概念 — 领域黑话交接手册

> 这份文档是给**刚接手 SmartQuality（内部也叫 Smart TPM）产线声学质量检测平台**的人准备的"黑话词典"。平台横跨"产线组织建模 → 检测流程编排 → 算法执行 → 数据集标注 → 测试与生产检测记录 → 声学采集子系统"六层，每一层都有一批只在内部口口相传的术语。看懂这一篇，后面所有系统文档、代码、MCP 工具名才不会一头雾水。

本文分三段读：
1. **概念分层总表** —— 一眼看全每个术语属于哪一层、大概是什么。
2. **逐层详解** —— 每个概念给出「定义 → 在系统里干什么 → 出现在哪个系统/仓库」三要素。
3. **关系图与延伸阅读** —— 把术语串成一张实体关系图，并指到更细的子百科。

术语首次出现即在括号里给出英文/字段名，后文直接复用。凡涉及部署地址、内网 IP、凭据的，本文一律回避，需要时请查仓库 `README.md` 与各变体配置。

---

## 第一段 · 概念分层总表

| 层 | 概念（中文 / 代码名） | 一句话 |
|---|---|---|
| 组织与产线 | 组织 / `company`（`companyId`） | 租户单位，所有数据按它硬隔离 |
| 组织与产线 | 项目 / `project` | 一次交付/一条产线的业务容器 |
| 组织与产线 | 产品 / `product` | 项目下被检测的具体产品型号 |
| 组织与产线 | 工位 / `workstation` | 物理产线上的一个作业位 |
| 组织与产线 | 检测点 / `detect_point` | 工位上的一个物理检测位置 |
| 组织与产线 | 阶段 / `detect_stage` | 检测点内的时序步骤 |
| 流程与算法 | 检测流程 / `detect_flow` | 一个阶段实际跑的算法工作流（版本化） |
| 流程与算法 | 流程节点 / `detect_flow_node` | 检测流程图里的一个节点 |
| 流程与算法 | 流程执行 / `detect_flow_execution` | 检测流程跑一次留下的执行实例 |
| 流程与算法 | 算法 / `algorithm` | 单个算法的注册条目 |
| 流程与算法 | 算法流 / `algorithm_flow` | 多算法编排出的子工作流 |
| 流程与算法 | 算法参数 / `algorithm_param` | 可被多个流程复用的参数模板 |
| 数据与标注 | 数据集 / `dataset` | 标注样本的集合 |
| 数据与标注 | 样本 / `sample`（`dataset_item`） | 数据集里的一条数据 |
| 数据与标注 | 标记 / `marks`（粗标 / 细标） | 给样本打的标注 |
| 数据与标注 | 标注 schema / `annotationSchema` | 定义可打哪些标签的模板 |
| 数据与标注 | 标注状态 / `markCode` | 样本的标注完成度状态机 |
| 数据与标注 | 抽样比例 / `sampleRatio` | 从数据集抽多少比例进测试任务 |
| 测试与生产记录 | 测试任务 / `test_task` | 批量跑检测流程的一个批次 |
| 测试与生产记录 | 测试记录 / `test_record` | 测试任务里单条样本的执行结果 |
| 测试与生产记录 | 生产检测记录 / `detect_record` | 产线实时产生的检测记录（工位分库） |
| 测试与生产记录 | 检测通道 / `detect_channel` | 生产记录挂载的通道维度 |
| 测试与生产记录 | 链路追踪 / trace | 从一条记录反查到算法执行的三层链路 |
| 测试与生产记录 | 标注审计 / `DetectAnnotationHistory` | 谁在何时改了哪条标注 |
| 声学子系统 | 联网录音 / `net_audio`（SmartAudio） | 工业现场长时录音 + 按时间戳回放 |
| 声学子系统 | 音频质检 / audio-quality（`ai_api_server_v2`） | DAG 引擎驱动的声学算法后端 |
| 声学子系统 | 移动套装 / SmartQuality 客户端 | 现场用的桌面采集/标注/上传一体应用 |

---

## 第二段 · 逐层详解

### A. 组织与产线层级

这一层把"现实世界的一条产线"翻译成数据库里的树。是所有查询的定位骨架——几乎每张表都挂在这棵树的某个节点下。

- **组织（company）**
  - **定义**：租户单位，一个 `companyId` 代表一家使用平台的公司/事业部。
  - **作用**：数据隔离的最高边界，**所有查询必带 `companyId`**，跨组织数据在存储层就看不见。
  - **出现在**：后端 Web 平台（`smart_tpm_web_v2`）几乎所有表；MCP 工具在鉴权时把用户绑定到其 `companyId`。

- **项目（project）**
  - **定义**：一次交付或一条产线对应的业务容器。
  - **作用**：把产品、工位、数据集、检测流程归拢到同一业务范围下。
  - **出现在**：Web 平台；MCP 元数据反查工具（`projects`）。

- **产品（product）**
  - **定义**：项目下被检测的具体产品型号。
  - **作用**：检测配置、数据集、算法参数往往按产品区分——同一条流程换个产品可能换一套参数。
  - **出现在**：Web 平台；MCP `products` / `project_products_list`。

- **工位（workstation）**
  - **定义**：物理产线上的一个作业位。
  - **作用**：用户的**可见性按工位组装**——你能看到哪些数据，取决于你被授权了哪些工位。
  - **出现在**：Web 平台（`WorkstationService`）；MCP `workstations_list`。

- **检测点（detect_point）**
  - **定义**：工位上的一个物理检测位置（"在哪儿测"）。
  - **作用**：三层时序结构的第一层，一个检测点下可以有多个阶段。
  - **出现在**：Web 平台；MCP `detect_points_list`。

- **阶段（detect_stage）**
  - **定义**：检测点内部的时序步骤（"第几步测"）。
  - **作用**：**一个阶段绑一个检测流程**——阶段是"业务时序"与"算法流程"之间的挂载点。
  - **出现在**：Web 平台；MCP `detect_station_stages_list`。

> 记忆口诀：**检测点（detect_point）→ 阶段（detect_stage）→ 检测流程（detect_flow）** 是三层时序结构，从"物理位置"一路收敛到"实际跑的算法图"。

### B. 检测流程与算法层

这一层是平台的"引擎室"：把一堆算法编排成图，版本化后交给产线跑。注意**检测流程**和**算法流**是两套解耦的编排，别混。

- **检测流程（detect_flow）**
  - **定义**：一个阶段实际执行的算法工作流的"图"。
  - **作用**：产线跑的是某个已发布（published）版本；流程被**版本化**（`detect_flow_version`），改流程 = 出新版本，老记录仍指向老版本。
  - **出现在**：Web 平台流程编排页；MCP `detect_flows_list/get/list_nodes`。细粒度字段见子百科 [detect-flow-node-reference](../../detect-flow-node-reference/SKILL.md)。

- **流程节点（detect_flow_node）**
  - **定义**：检测流程图里的一个节点，类型有 `algorithm / merge / branch / output`。
  - **作用**：`algorithm` 节点绑一条算法流；`merge` 节点把多个上游模型结果合并成最终结论（含 `show_model_result` / `skip_model_result` / `add_tag` 三个开关）。节点级配置放在 `currentNodeAttr`（JSON）。
  - **出现在**：Web 平台；MCP `detect_flows_list_nodes`。

- **流程执行（detect_flow_execution）**
  - **定义**：检测流程在某条记录上跑一次留下的执行实例。
  - **作用**：把"流程定义"和"这一次实际发生了什么"关联起来，是链路追踪的中间层。
  - **出现在**：后端执行引擎；MCP `detect_flow_executions_*`、`detect_logs_*`。

- **算法（algorithm）**
  - **定义**：单个算法的注册条目（一个可调用的检测能力）。
  - **作用**：算法本身是最小复用单元，被算法流引用。
  - **出现在**：Web 平台；MCP `algorithms_list/get`。

- **算法流（algorithm_flow）**
  - **定义**：多个算法编排出来的子工作流，**与检测流程解耦**。
  - **作用**：让"算法侧的编排"独立演化——检测流程节点通过 `algorithmFlowId` 引用某条算法流的某个版本。
  - **出现在**：Web 平台；MCP `algorithm_flows_*`。

- **算法参数（algorithm_param）**
  - **定义**：算法的参数模板（阈值、开关等）。
  - **作用**：可被多个检测流程引用；同一算法在不同产品上常靠换参数适配。
  - **出现在**：Web 平台；MCP `algorithm_params_search`。

### C. 数据集与标注层

这一层是"喂给算法的训练/评测燃料"，也是 AI 打标能力的落点。粗标 / 细标是最常被问混的一对。

- **数据集（dataset）**
  - **定义**：标注样本的集合，挂在项目 / 产品下。
  - **作用**：训练与评测算法的数据来源；测试任务从数据集里抽样本来跑。
  - **出现在**：Web 平台；MCP `datasets_list/get`、`datasets_query_samples`。字段细节见 [dataset-fields-reference](../../dataset-fields-reference/SKILL.md)。

- **样本（sample / dataset_item）**
  - **定义**：数据集里的一条数据（一个媒体 + 它的标注）。
  - **作用**：标注、抽样、检测的最小对象。
  - **出现在**：Web 平台；MCP `datasets_query_samples`、`datasets_get_annotations`。

- **标记（marks）：粗标 与 细标**
  - **定义**：给样本打的标注，分两层——
    - **粗标（rough / coarse annotation）**：**样本级**，整张图/整条记录一个结论（如 `OK / NG / unknown`）。
    - **细标（detail / fine annotation）**：**ROI 级**（Region of Interest，感兴趣区域），在一张图里框出多个区域各自标注。
  - **作用**：粗标定"整体判定"，细标定"哪一处出了问题"；两者共同构成算法的监督信号。AI 打标能力（写工具 `marks_set_rough` / `marks_add_detail` 等）就是往这两层写数据。
  - **出现在**：Web 平台标注页；MCP 标记系统读写工具（`rough_marks_list` / `detail_marks_list` / `marks_*` 写）。任务型 skill 见 [mark-samples-with-ai](../../mark-samples-with-ai/SKILL.md)。

- **标注 schema（annotationSchema）**
  - **定义**：数据集上的 JSON 模板，定义粗标 / 细标可选哪些 label、哪些字段必填。
  - **作用**：约束打标只能用字典内的合法标签；AI 打标前须先取字典（`project_marks_list`）再用 `markCode`。
  - **出现在**：数据集主表；MCP `project_marks_list`。

- **标注状态（markCode）**
  - **定义**：样本上的标注完成度状态机：`unmarked → coarse_done → fine_done → locked`。
  - **作用**：`fine_done` 自动含粗标完成；`locked` 是**终态**，人工锁定后任何写工具都不再改它。常与 `sampleRatio` 组合过滤（如"细标完成 + 抽 50%"）。
  - **出现在**：样本表；MCP 数据集写工具遵守其锁定语义。

- **抽样比例（sampleRatio）**
  - **定义**：0~1 的浮点数，表示从数据集里按此比例抽样进入测试任务。
  - **作用**：数据集层是默认值，被测试任务复制为初始值后可覆盖。`0.3` = 抽 30%。
  - **出现在**：数据集主表、测试任务；见 [dataset-fields-reference](../../dataset-fields-reference/SKILL.md)。

### D. 测试与生产检测记录层

同样是"检测记录"，但分两个世界：**测试任务**是"离线批量重跑验证"，**生产检测记录**是"产线实时真枪实弹产生的"。两者存储与用途都不同。

- **测试任务（test_task）**
  - **定义**：一次批量跑检测流程的批次。
  - **作用**：用数据集样本离线验证一版检测流程/算法效果；`sampleRatio` 决定抽多少样本进来。
  - **出现在**：Web 平台；MCP `test_tasks_list/get/create_from_template`。基于历史任务建草稿见 [create-test-task-from-template](../../create-test-task-from-template/SKILL.md)。

- **测试记录（test_record）**
  - **定义**：测试任务执行后单条样本的产物，对应一个 `detect_flow_execution`。
  - **作用**：逐条查看某样本在该版流程下跑出的结果，是回溯算法链路的起点。
  - **出现在**：Web 平台；MCP `test_records_list/get`。定位算法链见 [locate-test-record-algorithm-chain](../../locate-test-record-algorithm-chain/SKILL.md)。

- **生产检测记录（detect_record）与工位分库**
  - **定义**：产线运行时实时产生的检测记录。
  - **作用**：**工位分库**——生产记录量大，按工位/通道维度分库分表存放，查询时要带对分库键；这是它区别于测试记录的关键工程约束。
  - **出现在**：生产检测库；MCP `detect_records_list/get/ping`。查询任务见 [query-detect-records](../../query-detect-records/SKILL.md)。

- **检测通道（detect_channel）**
  - **定义**：生产检测记录挂载的通道维度（`channel_id`）。
  - **作用**：一键链路追踪的入口——`detect_records_trace(channel_id=...)` 可一次拿回三层链路。
  - **出现在**：生产检测库；MCP `station_channel_list`、`detect_records_trace`。

- **链路追踪（trace）**
  - **定义**：从一条检测记录反查到"检测流程执行 → 各阶段 → 各节点算法调用"的三层链路。
  - **作用**：复现客户报错、定位是哪一步算法判错的核心手段。
  - **出现在**：MCP `detect_records_trace` + `detect_logs_executions/stages/nodes_*`；复现流程见 [reproduce-customer-error](../../reproduce-customer-error/SKILL.md)。

- **标注审计（DetectAnnotationHistory）**
  - **定义**：记录每次标注写操作的审计表。
  - **作用**：所有 `marks_*` 写工具自动写这张表，`createBy` / `updateBy` 是被授权的真实用户 ID；可反查"AI 哪次会话改了什么"。
  - **出现在**：后端；MCP `annotation_history_list`。

### E. 声学质检子系统

平台的检测对象核心是**声学信号**（产品运转的声音、振动等）。声学侧分三块，各有独立仓库与独立子百科。

- **联网录音（net_audio，产品/服务名 SmartAudio）**
  - **定义**：工业现场的联网录音系统，采集端 + 管理端双进程架构。
  - **作用**：现场长时录音 + 按时间戳精准回放 + 给上游算法直接拉取指定阶段的音频字节。物理麦克风按 A/B/C/D 四通道映射。
  - **出现在**：独立硬件盒子 + 软件；详见子百科 [about-net-audio](../../about-net-audio/SKILL.md)。

- **音频质检后端（audio-quality，代码仓 `ai_api_server_v2`）**
  - **定义**：DAG（有向无环图）流程引擎驱动的声学算法后端。
  - **作用**：接收音频，跑 21+ 种 `BF_*` 算法（分类、评分、特征提取、异常/脉冲/电机检测、分贝测量等），双视角聚合出结论。它是检测流程里 `algorithm` 节点最终打到的算力。
  - **出现在**：GPU 集群；详见子百科 [about-audio-quality](../../about-audio-quality/SKILL.md)。

- **移动套装（SmartQuality 客户端）**
  - **定义**：现场使用的桌面应用（Electron），采集/标注/上传一体，内部称"移动套装"。
  - **作用**：现场人员用它录制、打标、上传样本；数据落到专用的 `smart_quality_client_sync` 同步库（比标准库多几列扩展字段）。
  - **出现在**：`SmartQuality-client` 仓库；同步链路见 [09-tech-stack.md](./09-tech-stack.md)。

---

## 第三段 · 实体关系图与延伸阅读

把上面的术语串成一棵树，就是平台的数据主干：

```
组织 (company)  ← companyId 硬隔离
 └─ 项目 (project)
     └─ 产品 (product)
         └─ 工位 (workstation)         ← 用户可见性按工位组装
             ├─ 检测点 (detect_point)          物理位置
             │   └─ 阶段 (detect_stage)        时序步骤
             │       └─ 检测流程 (detect_flow) 实际跑的算法图（版本化）
             │           └─ 节点 (detect_flow_node)
             │               └─ 算法流 (algorithm_flow) → 算法 (algorithm) + 参数
             │
             ├─ 测试任务 (test_task)           离线批量重跑
             │   └─ 测试记录 (test_record) → detect_flow_execution
             │
             └─ 生产检测记录 (detect_record)   实时、工位分库
                 └─ 检测通道 (detect_channel) → trace 三层链路

数据集 (dataset)                                 挂在 project / product 下
 └─ 样本 (sample) → 标记 (marks: 粗标 rough / 细标 detail)
     └─ markCode 状态机 · annotationSchema 字典 · sampleRatio 抽样

声学子系统：net_audio（采集）→ audio-quality/ai_api_server_v2（算法）
移动套装 SmartQuality 客户端（现场采集/标注/上传）
```

**这些术语在两个"世界"里各有一套接口**：Web 平台（人点界面）和 MCP 插件（AI 用自然语言查）。同一个概念，Web 上是页面，MCP 上是工具。

延伸阅读：

| 想深入 | 去这里 |
|---|---|
| 业务实体关系权威版 | [smart-tpm-business-concepts](../../smart-tpm-business-concepts/SKILL.md) |
| 数据集字段 / markCode / sampleRatio 细节 | [dataset-fields-reference](../../dataset-fields-reference/SKILL.md) |
| 检测流程节点字段（merge 三开关） | [detect-flow-node-reference](../../detect-flow-node-reference/SKILL.md) |
| 声学采集（net_audio / A-B-C-D 通道） | [about-net-audio](../../about-net-audio/SKILL.md) |
| 声学算法后端（DAG / BF_* 算法） | [about-audio-quality](../../about-audio-quality/SKILL.md) |
| MCP 插件怎么把这些暴露给 AI | [about-smart-tpm-mcp](../../about-smart-tpm-mcp/SKILL.md) |
| 各系统技术栈 | [09-tech-stack.md](./09-tech-stack.md) |
| 插件生态与子百科地图 | [12-ecosystem.md](./12-ecosystem.md) |

> 术语与代码字段可能随版本演进；本文与代码/子百科冲突时，**以代码和对应子百科为准**。

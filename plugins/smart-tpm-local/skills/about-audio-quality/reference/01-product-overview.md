# 01 · 产品概览

> 事实来源：`README.md`、`docs/product_architecture_doc.md`、`v2/` 代码。

## 定义：这是什么

**AI 音频质检系统**（代码仓 `ai_api_server_v2`）是面向**工业质检场景的音频 AI 推理服务**。典型用途：汽车零部件下线时录制声音，系统判定每个检测点位是合格（OK / positive）还是不合格（NG / negative）。

它的核心不是单个模型，而是一套**可视化编排 + 多模型协同**的引擎：

- 前端用流程编辑器画出 **DAG（有向无环图）**检测流程 → 导出 `{nodes, edges}` JSON
- 后端 **DAG 引擎**把流程拓扑排序后逐节点执行：切分音频 → 校验 → 多模型并行推理 → 结果合并 → 条件路由 → 输出结论
- 通过 `mode` 参数把推理动态分发到 **21+ 种算法模型**（分类 / 评分 / 特征提取 / 异常检测 / 分贝 / 脉冲 / 仲裁等）

对外只有一个推理入口：`POST /api/v2/detect/execute`，由 ERP / MES / MASS 等业务系统调用。

## 细节：核心能力

| 能力 | 说明 | 详见 |
|---|---|---|
| **DAG 流程引擎** | fork/join 并发、条件路由、嵌套子流程、节点启用/禁用 | [03-key-concepts.md](./03-key-concepts.md) |
| **检测引擎** | 解析检测通道（多工位/多阶段），并行调度算法子流程，双视角聚合判定 | [04-detect-engine.md](./04-detect-engine.md) |
| **多模型动态调度** | `mode` 参数动态分发到不同算法模型，无需改代码 | [05-algorithm-models.md](./05-algorithm-models.md) |
| **三级参数解析** | 前端配置 > 数据库缓存 > 本地 JSON 默认值，逐级回退 | [03-key-concepts.md](./03-key-concepts.md) |
| **热加载 / 延迟加载** | `reloadOne` 不停机更新流程与模型；模型首次调用时延迟加载 | [03-key-concepts.md](./03-key-concepts.md) |
| **多 GPU 集群** | 1 master + 4 worker（各占 1 GPU）+ nginx 反代，自动负载均衡 | [06-deployment-cluster.md](./06-deployment-cluster.md) |
| **执行日志追溯** | 三张 detect_log 审计表，全链路异常可追溯 | [04-detect-engine.md](./04-detect-engine.md) |

## 示例：能力边界（已支持 / 不支持）

### 已支持

音频质量检测、音频分类识别、多维度评分（dB/RMS/FFT）、音频特征提取、异常声音检测、脉冲/电机异常检测、分贝测量、多算法仲裁判定、双视角结果聚合、流程可视化编辑、模型热加载、多节点集群、实时日志监控、API 密钥鉴权、ONNX 模型导出（边缘部署用）。

### 目前不支持（弱措辞，状态见 [08-roadmap.md](./08-roadmap.md)）

| 功能 | 原因 |
|---|---|
| 视频流实时检测 | 当前仅支持离线音频文件分析，不支持实时流式输入 |
| 图像质检 | 算法层专注音频信号处理，未集成图像模型 |
| 自动模型训练 | 仅负责推理，训练通过 ClearML 平台完成 |
| 多语言 SDK | 当前仅提供 REST API，无 Java / C# 等 SDK |
| 消息队列异步 | 检测请求为同步响应，未接入 Kafka / RabbitMQ 异步回调 |

### 性能限制（典型值，来源 product_architecture_doc）

| 指标 | 数值 |
|---|---|
| 单请求内 stage 并行线程 | `DETECT_MAX_WORKERS`：代码默认 16，**生产部署设 8**（见 [06-deployment-cluster.md](./06-deployment-cluster.md) 与 CLAUDE.md） |
| 单流程 DAG 并行节点 | 最多 15 |
| 单次检测耗时 | 2~15 秒（取决于音频时长、算法复杂度、GPU） |
| 单节点 API 并发 | ~10~30 QPS（受 GPU 推理吞吐限制，集群横向扩展） |

> 术语首次出现见 [glossary/terms.md](../glossary/terms.md)：DAG、mode、point_first / stage_first、stageAssignments、BF_*。

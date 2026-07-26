# 场景：一条录音的端到端旅程

把散落在 6 个系统里的环节串成一条线：**一段产线录音，从被采集到最终被 AI 查询，中间经过谁、变成什么**。这是理解整个 SmartQuality 最有效的一条主线。分层架构见 [reference/02-architecture.md](../reference/02-architecture.md)。

## 全链路一图

```
① 采集           ② 上云归档         ③ 数据同步            ④ 检测/标注            ⑤ 分析/AI
产线现场          采集端 App          sync-client → server   Web 检测平台            Tinia / MCP
─────────        ──────────         ──────────────────    ────────────────      ─────────────
声学盒子/麦克风 → 录音判 OK/NG    →  本地攒"块" gRPC 上报 → 中心库 detect 记录   → Tinia 数据集分析
(SV30 四通道)     存本地+对象存储     校验去重写中心 MySQL     检测流程跑算法出结论      framework_v3 出日报
                                    音频分片传中心 MinIO     人工/ AI 粗标+细标        smart_quality_plugins
                                                                                  MCP 暴露给本地 AI 查询
```

## 逐段拆解

### ① 采集：现场录音，就地判定
- 现场用**声学盒子（SV30）**或麦克风采集音频；**采集端 App**（安卓 `smart_tpm_app` 或桌面 `SmartQuality-client`）录音、试听、就地跑算法给出 **OK/NG**，并把音频与结果存本地、写入待上传队列。
- 关键产物：一条**检测记录** + 一个音频文件（PCM/WAV）。
- 详见 [systems/04-app-client.md](../systems/04-app-client.md)。

### ② 上云归档：把结果和文件交给同步链路
- 采集端把"结果 jsonl + 音频副本"原子写入本机共享目录（sqMode 场景），与同步客户端**通过文件系统解耦**，不走网络耦合。
- 详见 [systems/03-data-sync-client.md](../systems/03-data-sync-client.md) 的 `sq_inbox` 约定。

### ③ 数据同步：从现场搬到中心
- **sync-client** 把数据行 + 关联文件打包成**块（block）**，经 gRPC 上报；音频大文件分片续传。
- **sync-server** 校验、三层去重、按 `client_id` 隔离写入中心 **MySQL**，音频落中心 **MinIO**，并透明重写文件 URL。
- 断网可攒、在线补发、重复不落重（`block_id` 幂等）。详见 [systems/02](../systems/02-data-sync-service.md) / [systems/03](../systems/03-data-sync-client.md)。

### ④ 检测与标注：在平台里变成可用数据
- 数据进入 **Web 检测平台**（smart_tpm_web_v2）：生产检测记录按**工位分库**落库；可回溯**链路追踪**（记录→流程执行→阶段→节点算法）。
- 样本进**数据集**，经**粗标（OK/NG）**与**细标（ROI 框选）**标注，状态机走到 `locked`；标注写操作全程审计（DetectAnnotationHistory）。
- 详见 [systems/05-web-platform.md](../systems/05-web-platform.md) 与 [reference/04-key-concepts.md](../reference/04-key-concepts.md)。

### ⑤ 分析与 AI：让数据产生价值
- **Tinia_nodes_smart_quality_app** 把录音接成 Tinia 数据集节点；**framework_v3_smart_quality** 上的 daily/plugins/apps 做分析与日报。
- **smart_quality_plugins**（本仓库）通过 **MCP** 把后端多个模块的工具暴露给本地 AI 客户端——工程师可用自然语言查检测数据、追算法链路、复现报错、辅助打标。
- 详见 [systems/06-mobile-suite.md](../systems/06-mobile-suite.md) 与子百科 [about-smart-tpm-mcp](../../about-smart-tpm-mcp/SKILL.md)。

## 这条线揭示的系统边界

| 环节 | 谁负责 | 交给下游的"接口物" |
|---|---|---|
| 采集判定 | 采集端 App | 检测记录 + 音频文件 |
| 现场→中心 | sync-client / server | 中心库里的行 + MinIO 里的音频 |
| 管理与标注 | Web 平台 | 数据集 + 标注 + 可追溯的检测记录 |
| 分析与暴露 | Tinia / framework_v3 / plugins | 日报 + AI 可查询的工具接口 |

> 反过来排障也走这条线：AI 查到某条记录异常 → 用 trace 反查流程执行与节点算法 → 定位是采集、同步还是算法环节的问题。跨系统联调时，先确认数据卡在这五段的哪一段。

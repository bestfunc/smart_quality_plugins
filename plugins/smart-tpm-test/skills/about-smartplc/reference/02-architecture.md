# 02 · 系统架构

> 事实来源:`BestPLC/` 代码树、`App.xaml.cs`、`Server/StableWorkflow.cs`、`PlcAdapter/`、`Server/SmartAudio|SmartQuality|Algorithm|Storage/`。

## 总览

SmartPLC 是一个**分层的单进程应用**:UI 在最上,往下是工作流引擎、协议适配层、对外集成层,旁路还有一组横切服务(日志 / 监控 / 追踪)。所有层跑在同一个 WPF 进程里,靠 `async/await` + 线程池支撑多工位并发。

## 分层架构

```
┌───────────────────────────────────────────────┐
│ UI 层 (WPF / XAML)                             │
│ MainWindow → NavigationView → 各 Page          │
│ (工位监控 / 告警日志 / 音频日志 / 数据采集)    │
├───────────────────────────────────────────────┤
│ 工作流引擎层 (Server/)                          │
│ StableWorkflow      ← 主工位:心跳/信号循环      │
│   └ StableWorkChildflow ← 子流程:单产品 8 阶段  │
├───────────────────────────────────────────────┤
│ 协议适配层 (PlcAdapter/)                        │
│ Siemens S7-1500 / Modbus TCP / 欧姆龙 FINS TCP       │
│ ProtocolFactory + 连接池 + 自动重连             │
├───────────────────────────────────────────────┤
│ 对外集成层 (Server/)                            │
│ SmartAudio盒子 / 算法引擎 / SmartQuality / MinIO│
├───────────────────────────────────────────────┤
│ 横切服务 (Services/)                            │
│ 告警日志 / GC 监控 / 数据库调用追踪 / 心跳记录  │
└───────────────────────────────────────────────┘
```

## 核心模块职责

| 模块 | 位置 | 职责 |
|---|---|---|
| **UI 框架** | `MainWindow.xaml.cs`、`Views/Pages/` | 主窗体、多工位导航、监控与日志展示 |
| **主工作流** | `Server/StableWorkflow.cs` | 心跳读取 → 信号解析 → 触发录音/算法 → 结果写回;管理子流程生命周期 |
| **子工作流** | `Server/StableWorkChildflow.cs` | 单产品 Ready→Running→Stop 状态机,8 阶段检测循环,动态点位读写,结果汇总写 PLC |
| **PLC 适配** | `PlcAdapter/Siemens|Modbus|Omron/` | 三协议客户端;`ProtocolFactory` 按配置选协议,`ProtocolConnectionManager` 管连接池/重连 |
| **音频集成** | `Server/SmartAudio/SmartAudioClient.cs` | 录音盒子启停、时间同步、流数据拉取 |
| **质检平台** | `Server/SmartQuality/SmartQualityClientV2.cs` | MySQL 直连:认证、工位/产品/阶段元数据、结果落库、灵敏度同步 |
| **算法引擎** | `Server/Algorithm/AlgorithmClient.cs` | HTTP/JSON 调 `/api/v2/detect/execute`,超时控制、结果解析 |
| **文件存储** | `Server/Storage/MinIOFileClient.cs` | 录音文件 S3 上传(bucket: audio) |
| **横切服务** | `Services/` | 告警拦截、GC 暂停监控、数据库调用追踪、心跳记录 |
| **配置/模型** | `Settings.settings`、`Models/` | 连接串、开关、点位与设备配置数据结构 |

## 关键数据流

一个产品从触发到出结果的主链路(详见 [05-detection-workflow.md](./05-detection-workflow.md)):

```
PLC 信号(每 50ms 轮询)
  → StableWorkflow 检测到 Ready → 新建 StableWorkChildflow
  → 逐阶段(1~8):写阶段号 → 触发录音(SmartAudioClient.Start)
  → 拉取音频 → 送算法(AlgorithmClient)→ 解析 OK/NG
  → 结果落库(SmartQualityClientV2)+ 写回 PLC(WPointStageResult)
  → 全部阶段完成 → 汇总 WPointResult → 子流程 Dispose
```

## 对外集成点

| 系统 | 协议 | 触发时机 | 入口类 |
|---|---|---|---|
| **SmartAudio 录音盒子** | HTTP + WebSocket | 阶段开始/结束 | `SmartAudioClient` |
| **算法引擎** | HTTP/JSON + X-API-Key | 拿到音频后 | `AlgorithmClient` |
| **SmartQuality(MySQL)** | SQL + 事务,直连 | 元数据读取、结果落库 | `SmartQualityClientV2` |
| **MinIO** | S3(HTTP/HTTPS) | 音频持久化 | `MinIOFileClient` |
| **产线 PLC** | S7 / Modbus / 欧姆龙 | 每 50ms 轮询 | `PlcAdapter/*` |

集成细节(地址、开关、数据格式)见 [06-integrations.md](./06-integrations.md)。

## 设计要点

- **数据库直连而非 HTTP API**:`SmartQualityClientV2` 直连 MySQL,换来事务一致性(多阶段结果原子写入)与更低延迟——在 v3.5 高频调用场景下是关键取舍。
- **多工位并发**:每工位一个 `StableWorkflow`,子流程按产品创建/销毁;`App` 启动时 `ThreadPool.SetMinThreads(50,50)` 降低异步起调延迟。
- **可观测优先**:GC 暂停监控 + 告警拦截 + 数据库调用追踪,为现场性能诊断兜底(配套开发类 skill `smartplc-perf`)。

## 延伸阅读

- 协议适配怎么做的 → [04-plc-protocols.md](./04-plc-protocols.md)
- 技术栈与依赖 → [07-tech-stack.md](./07-tech-stack.md)
- 名词不熟 → [03-key-concepts.md](./03-key-concepts.md)

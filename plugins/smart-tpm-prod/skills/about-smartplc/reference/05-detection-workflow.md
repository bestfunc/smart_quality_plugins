# 05 · 检测工作流

> 事实来源:`BestPLC/Server/StableWorkflow.cs`、`BestPLC/Server/StableWorkChildflow.cs`、`BestPLC/Settings.settings`(默认值)、`BestPLC/Docs/UDP信号处理流程分析.md`。

## 概览:主流程 vs 子流程

检测工作流是 SmartPLC 的心脏:把"PLC 信号"翻译成"一次完整检测",再把"OK/NG 结果"翻译回"PLC 信号"。它分两层——**主流程常驻、子流程按产品生灭**。

| 维度 | 主流程 `StableWorkflow` | 子流程 `StableWorkChildflow` |
|---|---|---|
| **粒度** | 一个工位一个,常驻整个运行期 | 一个产品一次,检测完即销毁 |
| **职责** | 心跳/信号轮询、产品号校验、子流程生命周期管理 | 单产品 8 阶段检测状态机、录音/算法/落库/写回 |
| **驱动** | 三个定时器(主轮询 50ms / 心跳 300ms / 维护 1s) | 一个定时器(50ms 轮询控制点) + 事件回调 |
| **状态** | 心跳值、读心跳健康度、当前子流程引用 | `mainRunStatus`(主)/ `stageRunStatus`(阶段)双状态机 |

主流程是"门卫 + 调度",子流程是"流水线工人"。一个产品来 → 主流程开一个子流程;产品走 / 超时 / 复位 → 子流程 `Dispose`。术语见 [03-key-concepts.md](./03-key-concepts.md)。

## 主流程 StableWorkflow

主流程在 `Run()` 里起三个 `System.Threading.Timer`,各司其职、互不阻塞(分离是为降低心跳被拖慢的概率):

| 定时器 | 周期(默认) | 干什么 |
|---|---|---|
| **主轮询** `TimerCallback` | `PollingInterval` = 50ms | 读控制点(主开关 / 复位)、检测超时、同步子流程状态、可选自动测试 |
| **心跳写** `HeartbeatWriteCallback` | `HeartbeatInterval` = 300ms | 向 PLC 写自增心跳(0-9 循环),证明中控存活;最小化耗时 |
| **维护** `MaintenanceCallback` | 1000ms | PLC 断线重连、**产品号变化检测**、读 PLC 回写心跳并对账 |

**产品号变化检测**(`WProductnumberTimerCallback`):每秒读一次产品号点位,经 `NormalizeProductNumber` 归一化(截掉 S7 STRING 尾部脏字节)后与上次值比对,变了才回写 PLC——避免每秒无谓写入。

**子流程生命周期**(主轮询里处理控制信号):

```
主开关上升沿 (0→1) → OnMainStart()
  ├─ 读产品号 → 按 (productNumber, stationID) 查 productId
  │    └─ EnableProductValidation=true 且查无 → 写错误码, 不开检测
  │       否则查无 → 自动建产品记录
  ├─ new StableWorkChildflow(..., productId)  // 创建子流程
  ├─ 读条码(按长度截取) + 当前阶段, 登记 ChildWorkflowInfo(最多留 5 条)
  └─ childflow.Run() + childflow.OnStart()

复位信号上升沿 → OnMainReset() → childflow.Dispose() → 置空引用
超时(> OperationTimeOut) → 落库 "Timeout" → OnMainReset(写 NG 兜底)
```

健康判定:`IsHealthy = PLC 已连 && 写心跳 ≥ 0 && 读心跳 5 秒内有变化`。

## 子流程 StableWorkChildflow

子流程是**事件驱动的双状态机**。`mainRunStatus` 走 `PreStart → Running → PreStop → Stop`;`stageRunStatus` 在每个阶段内独立走同一组状态。它自己也起一个 50ms 定时器,只轮询三个控制点:

| 控制点 | 上升沿语义 | 触发动作 |
|---|---|---|
| `RPointMainSwitch` | 跌到停止值(0/-1) | `OnStop()` — 结束整次检测 |
| `RPointStageSwitch` | 升到阶段值 | `OnStageStart()` — 开一个阶段 |
| `RPointStageSwitch` | 跌到停止值 | `OnStageStop()` — 结束当前阶段 |

`OnStart()` 里 `Start()` 调 `CreateDetectMain` 在数据库建主检测记录(拿到 `runningDetectMainID`),写 `WPointReady=1` 告诉 PLC"我准备好了,可以送阶段信号"。每个阶段的录音、算法、落库在并行 `Task` 中处理,不阻塞主轮询(见下)。

> 子流程持有自己的 `SmartQualityClientV2` + 多个 `SmartAudioClient`(每个录音盒子一个),并写一份独立的 `_instanceId` 日志目录,便于按产品追溯。

## 8 阶段时序

一次检测被切成 1-8 个阶段,每阶段由 PLC 的 `RPointStageSwitch` + `RPointCurrentStage` 驱动,独立录音 + 独立算法判定。单个阶段的动作链:

```
PLC 写阶段号 + 升 StageSwitch
  → OnStageStart()
      ├─ RefreshContext() 读上下文(条码/产品号/阶段号)
      ├─ ResetStage() 复位本阶段结果点位为 -1
      ├─ StartStage(): CreateDetectStage 建阶段记录 → 各盒子并行 Start() 录音
      │    (启用霍尔时同步 ModbusHallReader.StartRead())
      └─ ReadyStatusOn() 写 StageReady, 告诉 PLC 本阶段已就绪

PLC 跌 StageSwitch
  → OnStageStop()
      ├─ StopStage(): DetectStageStop + 各盒子并行 Stop()(拉音频/送算法)
      │    读霍尔数据 → 导出 CSV
      └─ 并行 Task(不阻塞主轮询):
            ├─ 算法直调开关开 → TryProcessStageStopDirectAsync 接管
            ├─ WaitingChannelDetectResult → WriteStageChannelResult(声道级动态点位)
            └─ WaitingStageResult → WriteStageResult(WPointStage{N}Result: 1=OK / 2=NG)
```

阶段号到点位的映射写死在 `WriteStageResult` 的 `switch`(stage "1".."8" → `WPointStage1Result`..`WPointStage8Result`)。等结果用轮询:每 50ms 查一次数据库,最长等 `WaitAlgorithmResultTime`(默认 5s),超时写兜底 `Code=50/timeout`。

**阶段触发模式可配**(`StageStartMode` / `StageReadyMode`,默认 `Fixed1`):

| 模式 | 含义 |
|---|---|
| `Fixed1` | 信号固定写 1 触发,阶段号另从 `RPointCurrentStage` 读 |
| `阶段数` | 信号值本身即阶段号(1-9),就绪也回写阶段号 |

## 结果聚合

所有阶段跑完、主开关跌停 → `OnStop()` → `Stop()` 做整次检测的收尾与聚合:

1. `SaveDetectMainOldResult` + `DetectMainStop` 在数据库标记检测结束。
2. fire-and-forget 后台 `Task`:`WaitingResult` 轮询主结果(`status` 为 `end/stopped/succeed/failed` 或 `result` 非空即返回)。
3. **先等所有阶段动态点位写完**(`_pendingStageResultTasks`),保证综合结果不抢在阶段值之前落 PLC。
4. `WriteMainDynamicResult` 写主声道级动态点位 + `WriteStageChannelResult` 用 `mainRes.channels` 兜底覆盖各阶段 db 值。
5. `WriteResult` 写综合结果 `WPointResult` + `WPointErrorCode`,再写 `WPointResultNotify=1` 通知 PLC 读取。

> 写回值约定:`-1` = 未就绪/复位,`1` = OK,`2` = NG。开关 `AlgorithmResultAlwaysOk=true` 时,落库保留真实结果,但 `WPointResult` 强制写 1(用于产线放行调试)。

## 异常 / 超时 / 重置处理

| 情况 | 触发点 | 处理 |
|---|---|---|
| **操作超时** | 主轮询发现 `Running` 且 `> OperationTimeOut`(默认 30s) | 落库 `Timeout` → `OnMainReset(写 NG 兜底)` → `WPointSystemStatus` 闪 1 持续 1s 后回 -1 |
| **现场复位** | `WPointReset` 上升沿 | 落库 `Reset` → `OnMainReset` → 系统状态闪 1 |
| **PLC 断连** | 主轮询 / 心跳 / 维护 | 心跳与主轮询静默跳过,由维护定时器 1s 周期重连;心跳值不复位避免恢复后重复 |
| **算法结果超时** | `WaitingStageResult` / `WaitingResult` 等满 5s | 写兜底 `Code=50/timeout`,不卡住流程 |
| **未启用算法** | `EnableAlgorithm=false` | 阶段/主结果落库 `Test`,PLC 仍写默认 `Code=100`,算法流程后台照走 |
| **检测中销毁** | `Dispose` 期间 fire-and-forget 任务 | `_isDisposed` 守卫:已销毁则跳过 PLC 写,防覆盖已复位点位 |

`Dispose` 会先停定时器,再 `AwaitPendingResultTasks` 等所有结果任务收尾(带超时),最后导出时序 CSV——确保结果不丢、点位不脏。

## 延伸阅读

- 整体分层与数据流 → [02-architecture.md](./02-architecture.md)
- 工位/检测点/阶段/灵敏度等概念 → [03-key-concepts.md](./03-key-concepts.md)
- 录音盒子 / 算法引擎 / MySQL 落库的对接细节 → [06-integrations.md](./06-integrations.md)
- 点位模型与 PLC 协议 → [04-plc-protocols.md](./04-plc-protocols.md)
- 完整术语 → [../glossary/terms.md](../glossary/terms.md)

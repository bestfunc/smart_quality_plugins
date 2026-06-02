# 术语表 · Glossary

> 事实来源:`../reference/03-key-concepts.md`(主),并汇总 01/02/04/05/06/07 各篇首次出现的专有名词与缩写。本页只做"查词",概念之间的关系与时序见对应 reference。

这是 SmartPLC(代码库 `BestPLC`)的统一术语词典。读代码、配置、文档或对接客户时,凡遇到下面这些名词,以本页解释为准;每条都链回最相关的 reference。术语按 **业务实体 / 流程 / 检测参数 / 技术与协议 / 集成系统** 分组,末尾附 **缩写与英文对照表**。

---

## 一、业务实体

| 术语 | 一句话解释 | 相关 reference |
|---|---|---|
| **检测工位**(detect_station) | 产线上一个检测位置,对应一套 PLC + 设备配置;数据库表 `detect_station`,代码里绑 `PlcConfig` | [03](../reference/03-key-concepts.md)、[06](../reference/06-integrations.md) |
| **检测点**(detect_point) | 工位内一个具体检测位,绑定某个智能设备的某个通道,带灵敏度;表 `detect_point`、`Models/DetectPoint.cs` | [03](../reference/03-key-concepts.md) |
| **智能设备**(smart_device) | 录音盒子。一个 IP + 多个声道(通道),挂在工位下;表 `smart_device`,带 `token` | [03](../reference/03-key-concepts.md)、[06](../reference/06-integrations.md) |
| **通道**(channel) | 录音盒子内部声道号(1-4),区分多路音频;直接对应 `detect_point.channel` | [03](../reference/03-key-concepts.md) |
| **产品**(product) | 被检测的产品型号。检测点配置、灵敏度按产品(`productId`)区分;表 `product` | [03](../reference/03-key-concepts.md) |
| **条码**(barcode) | 单个工件实例的唯一标识,用于数据追溯;`ChildWorkflowInfo.Barcode`,主流程按长度截取读入 | [03](../reference/03-key-concepts.md)、[05](../reference/05-detection-workflow.md) |
| **产品号**(productNumber) | PLC 上报的产品型号编码,经 `NormalizeProductNumber` 归一化后按 `(productNumber, stationID)` 查到 `productId` | [05](../reference/05-detection-workflow.md) |

> 实体层级:`检测工位 → 智能设备(盒子) → 通道 → 检测点`;产品决定该工位用哪套检测点配置。关系图见 [03-key-concepts.md](../reference/03-key-concepts.md)。

---

## 二、流程概念

| 术语 | 一句话解释 | 相关 reference |
|---|---|---|
| **主流程**(StableWorkflow) | 一个工位的常驻引擎:三个定时器周期读 PLC 心跳/信号,驱动子流程;`Server/StableWorkflow.cs` | [05](../reference/05-detection-workflow.md) |
| **子流程**(StableWorkChildflow) | 单个产品的一次完整检测,8 阶段状态机,检测完即 `Dispose`;`Server/StableWorkChildflow.cs` | [05](../reference/05-detection-workflow.md) |
| **阶段**(stage) | 一次检测内的步骤划分(1-8),每阶段独立录音 + 算法判定;结果点位 `WPointStage{1..8}Result` | [03](../reference/03-key-concepts.md)、[05](../reference/05-detection-workflow.md) |
| **主状态机**(mainRunStatus) | 子流程主状态:`PreStart → Running → PreStop → Stop` | [05](../reference/05-detection-workflow.md) |
| **阶段状态机**(stageRunStatus) | 每个阶段内独立走同一组状态,与主状态机并行 | [05](../reference/05-detection-workflow.md) |
| **控制点 / 信号点** | PLC 上驱动流程的读写位,如主开关、复位、阶段开关;上升/下降沿触发动作 | [05](../reference/05-detection-workflow.md) |
| **心跳**(heartbeat) | 中控向 PLC 写的自增值(0-9 循环),证明中控存活;默认 300ms 写一次 | [05](../reference/05-detection-workflow.md) |
| **结果聚合** | 所有阶段跑完后汇总主结果 `WPointResult` + 错误码,写 `WPointResultNotify=1` 通知 PLC | [05](../reference/05-detection-workflow.md) |
| **静态点位** | 配置里固定的读写点(心跳、信号、结果回写位),`ProtocolFactory.CreatePoint` 创建 | [04](../reference/04-plc-protocols.md) |
| **动态点位**(dynamic point) | 运行时按流程节点/阶段/通道写入 PLC 的点,支持值变换表达式;`Models/PlcConfigDynamicPoint.cs` | [03](../reference/03-key-concepts.md)、[04](../reference/04-plc-protocols.md) |
| **值变换**(valueTransform) | 动态点位写入前的单运算缩放/平移表达式,如 `*100`;`StableWorkChildflow.ApplyValueTransform` | [04](../reference/04-plc-protocols.md) |
| **阶段触发模式**(StageStartMode / StageReadyMode) | 阶段信号语义,默认 `Fixed1`(固定写 1 触发);另一档为信号值即阶段号 | [05](../reference/05-detection-workflow.md) |

> 上升沿 / 下降沿:PLC 信号从 0→1(上升)或跌到停止值(下降)的瞬间,SmartPLC 据此触发开始/结束动作,而非持续电平。

---

## 三、检测参数与判定

| 术语 | 一句话解释 | 相关 reference |
|---|---|---|
| **灵敏度**(sensitivity) | 检测点的音频检测灵敏度,DB 为 `smallint`,范围 1-255、默认 220;按产品下发给盒子 | [03](../reference/03-key-concepts.md) |
| **OK / NG** | 良品 / 不良品判定结果。写回 PLC 约定:`1`=OK、`2`=NG、`-1`=未就绪/复位 | [01](../reference/01-product-overview.md)、[05](../reference/05-detection-workflow.md) |
| **算法引擎** | 音频检测算法后端,接收音频返回 OK/NG;端点 `POST /api/v2/detect/execute` | [03](../reference/03-key-concepts.md)、[06](../reference/06-integrations.md) |
| **dataContent** | 算法请求体里的核心嵌套结构,含 `detect[]`/`detect_channel[]`/`detect_point[]`/`detect_station_stage[]`/`file[]` | [03](../reference/03-key-concepts.md)、[06](../reference/06-integrations.md) |
| **mode**(算法模式) | 算法调试开关:`"0"`=实际、`"1"`=全 OK、`"2"`=全 NG(后两者仅调试) | [06](../reference/06-integrations.md) |
| **point_first** | 算法响应的结果视图,`productResult` + `pointResults[].stageResults[]`,调用方据此路由回各通道 | [06](../reference/06-integrations.md) |
| **stageKey** | 算法响应中阶段结果的键,形如 `{pointKey}__stage{N}`,按 (PointKey, StageIndex) 路由 | [06](../reference/06-integrations.md) |
| **霍尔信号**(Hall) | 霍尔传感器脉冲,经 Modbus 读取,用于精确同步产品位置;`ModbusHallReader` | [03](../reference/03-key-concepts.md)、[07](../reference/07-tech-stack.md) |
| **兜底结果** | 算法超时(等满默认 5s)或未启用算法时写入的占位结果,如 `Code=50/timeout`、`Code=100`,不卡流程 | [05](../reference/05-detection-workflow.md) |

---

## 四、技术与协议

| 术语 | 一句话解释 | 相关 reference |
|---|---|---|
| **PLC** | 可编程逻辑控制器,产线设备控制核心;SmartPLC 作为中控读写它的信号点 | [01](../reference/01-product-overview.md)、[04](../reference/04-plc-protocols.md) |
| **Siemens S7-1500** | 西门子 S7 系列 PLC,经 S7 协议通信;驱动类 `SiemensS7Net(S1500)`,默认端口 102 | [04](../reference/04-plc-protocols.md) |
| **Modbus TCP** | 通用工业寄存器协议,常用于第三方设备/仪表;驱动类 `ModbusTcpNet`,默认端口 502 | [04](../reference/04-plc-protocols.md) |
| **欧姆龙(FINS TCP)** | OMRON CJ/CS/CP 系列走的协议;驱动类 `OmronFinsNet`,默认端口 9600 | [04](../reference/04-plc-protocols.md) |
| **OmronCip**(命名注意) | 欧姆龙实现的文件 / 类名,历史命名;底层实际是 FINS TCP(`OmronFinsNet`),**不是** EtherNet/IP CIP | [04](../reference/04-plc-protocols.md)、[07](../reference/07-tech-stack.md) |
| **字节序**(byteOrder / ModbusByteOrder) | 多字节类型的高低位排列,四档:`ABCD`(大端)/`BADC`/`CDAB`(字交换,默认)/`DCBA`(小端) | [04](../reference/04-plc-protocols.md) |
| **IPlc&lt;T&gt;** | 协议无关的统一读写接口(`Connect`/`Read`/`Write`/`WriteAsync`);`PlcAdapter/IPlc.cs` | [04](../reference/04-plc-protocols.md) |
| **ProtocolFactory** | 工厂类:按 `protocol` 字符串选协议,造连接与点位(`CreatePoint`/`CreateDynamicPoint`) | [04](../reference/04-plc-protocols.md) |
| **MultiProtocolAdapter** | 适配器:把 `IPlc<IPlcPoint>` 包成主流程只认的 `IPlc<SiemensPlcPoint>`,运行时转点位 | [04](../reference/04-plc-protocols.md) |
| **ProtocolConnectionManager** | 连接管理:标准连接(每工位一条常驻)+ 动态连接池;`Timer` 每 30 秒巡检重连 | [04](../reference/04-plc-protocols.md) |
| **HslCommunication** | 私有工业通信 SDK(`HslCommunication.dll`),三协议统一压在它上;以 `Reference` 引入,非 NuGet | [04](../reference/04-plc-protocols.md)、[07](../reference/07-tech-stack.md) |
| **WPF / .NET 8** | 运行时与 UI 框架:`net8.0-windows`、`WinExe`、`UseWPF=true`;桌面单进程应用 | [07](../reference/07-tech-stack.md) |
| **Server GC** | `ServerGarbageCollection=true`,多工位并发吞吐优先的 GC 模式;运行时由 `GcMonitorService` 监控 | [02](../reference/02-architecture.md)、[07](../reference/07-tech-stack.md) |
| **WPF-UI / NavigationView** | Fluent 风格主题控件库(3.0.5),主窗体导航 `NavigationView` 来自它 | [07](../reference/07-tech-stack.md) |
| **GC 暂停监控** | `Services/GcMonitorService.cs` 持续监控垃圾回收停顿,现场性能诊断证据之一 | [02](../reference/02-architecture.md) |

---

## 五、集成系统与外部服务

| 术语 | 一句话解释 | 相关 reference |
|---|---|---|
| **SmartAudio**(录音盒子) | 挂在工位下的智能设备,采集音频;REST(`/api/v1/*`)+ WebSocket,入口类 `SmartAudioClient` | [06](../reference/06-integrations.md) |
| **stop_and_fetch** | 录音盒子算法直调接口,按时间窗截取片段(daemon 不停录),响应单/多通道;超时放宽到 3 分钟 | [06](../reference/06-integrations.md) |
| **算法直调链路** | `EnableDirectAlgorithmCall=true` 时走 `stop_and_fetch` → MinIO → 算法,替代旧 edge_api 上传链路 | [06](../reference/06-integrations.md) |
| **SmartQuality** | 质检平台数据库(`bestplc`),`SmartQualityClientV2` **直连 MySQL + 事务**,换事务一致性与低延迟 | [02](../reference/02-architecture.md)、[06](../reference/06-integrations.md) |
| **DatabaseHelper / 数据库调用记录** | DB 访问入口;每次方法调用经 `RecordMethodCallAsync` 把 SQL 操作留痕(可观测兜底) | [06](../reference/06-integrations.md) |
| **MinIO** | S3 兼容对象存储,持久化录音文件;bucket 默认 `audio`,入口类 `MinIOFileClient` | [06](../reference/06-integrations.md) |
| **时间同步**(PushTimeSync) | V3 模式下录音前向盒子推 Unix 纳秒时间戳,同一盒子 5 分钟内限流一次 | [06](../reference/06-integrations.md) |
| **edge_api**(旧链路) | 历史的录音上传 + 算法链路,现由算法直调链路替代(`EnableDirectAlgorithmCall` 切换) | [06](../reference/06-integrations.md) |
| **集成开关**(Settings.Enable*) | `EnableMicrophone`/`EnableAlgorithm`/`EnableDirectAlgorithmCall`/`EnableSensitivitySync`/`EnableModbusHall` 等 | [06](../reference/06-integrations.md) |

---

## 六、缩写与英文对照表

| 缩写 / 英文 | 全称 / 含义 | 备注 |
|---|---|---|
| **PLC** | Programmable Logic Controller，可编程逻辑控制器 | 产线设备控制核心 |
| **S7** | Siemens 通信协议族 | S7-1500 走 `SiemensS7Net(S1500)`，端口 102 |
| **FINS TCP** | Factory Interface Network Service over TCP，欧姆龙 PLC 通信协议 | 驱动类 `OmronFinsNet`，端口 9600 |
| **CIP** | Common Industrial Protocol（EtherNet/IP） | `OmronCip` 类名沿用，但**实际走 FINS TCP，非 CIP** |
| **Modbus TCP** | Modbus over TCP/IP，通用寄存器协议 | `ModbusTcpNet`，端口 502 |
| **OK / NG** | OK = 良品 / NG（No Good）= 不良品 | 写回 PLC：1=OK、2=NG、-1=未就绪 |
| **GC** | Garbage Collection，垃圾回收 | 启用 Server GC，`GcMonitorService` 监控停顿 |
| **SDK** | Software Development Kit，软件开发工具包 | 指私有 `HslCommunication.dll` |
| **X-API-Key** | 算法引擎 HTTP 鉴权头 | 形如 `stpm-{env}-{hex}`，配置项 `AlgorithmApiKey` |
| **X-Token** | 录音盒子 REST 鉴权头 | 取自 `smart_device.token` |
| **WPoint** | Write Point，写回 PLC 的点位前缀 | 如 `WPointReady`/`WPointResult`/`WPointStage{N}Result` |
| **RPoint** | Read Point，从 PLC 读取的控制点前缀 | 如 `RPointMainSwitch`/`RPointStageSwitch`/`RPointCurrentStage` |
| **valueTransform** | 动态点位值变换表达式 | `*N`/`/N`/`+N`/`-N`，整数寄存器四舍五入取整 |
| **dataContent** | 算法请求体核心嵌套结构 | `detect`/`detect_channel`/`detect_point`/`file` 等 |
| **REST** | Representational State Transfer，HTTP 接口风格 | 录音盒子 `/api/v1/*`，用 RestSharp |
| **WS / WebSocket** | 全双工实时通信协议 | 录音盒子实时听音 `ws://{IP}:{Port}/api/v1/stream/live` |
| **S3** | Amazon S3 兼容对象存储协议 | MinIO 走 S3，`Minio` NuGet 包 |
| **DB** | Database，数据库 | 指 MySQL（`bestplc`），SmartQuality 直连 |
| **LTS** | Long-Term Support，长期支持版本 | 指 .NET 8 |
| **WPF** | Windows Presentation Foundation | 主 UI 框架 |
| **DTO** | Data Transfer Object，数据传输对象 | 算法/集成请求响应模型 |
| **ENV** | 运行环境标识 | `prod`（默认）/`dev`（录音走模拟） |
| **stage** | 阶段（1-8） | 一次检测内的步骤划分 |
| **channel** | 通道 / 声道（1-4） | 录音盒子内部声道号 |
| **HslCommunication** | 私有工业通信库 | 三协议统一底座 |

---

## 七、HslCommunication 驱动类速查

| 对外口径 | HslCommunication 驱动类 | 代码位置 |
|---|---|---|
| Siemens S7-1500 | `SiemensS7Net(SiemensPLCS.S1500)` | `PlcAdapter/Siemens/Siemens1500.cs` |
| Modbus TCP | `ModbusTcpNet` | `PlcAdapter/Modbus/ModbusTCP.cs` |
| 欧姆龙(FINS TCP) | `OmronFinsNet` | `PlcAdapter/Omron/OmronCip.cs` |

> 三协议**同源**:都调 HslCommunication 对应驱动,不是各自独立实现。详见 [07-tech-stack.md](../reference/07-tech-stack.md) §④。

---

## 延伸阅读

- 概念之间的关系与层级图 → [03-key-concepts.md](../reference/03-key-concepts.md)
- 协议 / 点位 / 字节序细节 → [04-plc-protocols.md](../reference/04-plc-protocols.md)
- 检测全流程时序(主/子流程、8 阶段) → [05-detection-workflow.md](../reference/05-detection-workflow.md)
- 集成系统地址 / 开关 / 数据格式 → [06-integrations.md](../reference/06-integrations.md)
- 技术栈与 NuGet 依赖 → [07-tech-stack.md](../reference/07-tech-stack.md)

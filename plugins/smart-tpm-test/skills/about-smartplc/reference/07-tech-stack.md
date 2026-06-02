# 07 · 技术栈与第三方依赖

> 事实来源:`BestPLC/BestPLC.csproj`(TargetFramework + 全部 PackageReference + Reference)、`BestPLC/App.config`、各依赖在代码中的使用点(`PlcAdapter/`、`Server/`、`Helpers/`)。

## 一句话

SmartPLC 是一个 **.NET 8 / WPF 桌面应用**(`WinExe`,纯净单进程),工业 PLC 通信统一压在私有 SDK **HslCommunication.dll** 上,其余能力靠一小组成熟 NuGet 包(HTTP / 日志 / JSON / MySQL / S3 / 音频)拼装。依赖数量克制,没有重型框架。

## ① 运行时

| 项 | 值 | 说明 |
|---|---|---|
| **TargetFramework** | `net8.0-windows` | .NET 8(LTS),仅 Windows |
| **OutputType** | `WinExe` | 桌面应用,无控制台窗口 |
| **UseWPF** | `true` | 主 UI 框架 |
| **UseWindowsForms** | `true` | 与 WPF 混用(托盘 / 部分原生控件) |
| **PlatformTarget** | `AnyCPU` | 不锁定 x86/x64 |
| **ServerGarbageCollection** | `true` | Server GC,多工位并发下吞吐优先 |
| **Nullable / ImplicitUsings** | `enable` | 现代 C# 默认开启 |
| **出品信息** | Product=`SmartPLC`,Company=`巅峰表现`,Authors=`Bestfunc` | 见 [01-product-overview.md](./01-product-overview.md) |

> **Server GC** 是性能取舍的一环:多工位各自跑异步循环,Server GC 用多个后台堆减少停顿。GC 暂停在运行时还被 `Services/GcMonitorService.cs` 持续监控(见 [02-architecture.md](./02-architecture.md) 的"可观测优先")。

## ② UI 框架

| 包 | 版本 | 用途 |
|---|---|---|
| **WPF-UI** | 3.0.5 | 主题化控件库(Fluent 风格):`NavigationView`、主窗体导航、各 Page |
| **WPF-UI.Tray** | 3.0.5 | 系统托盘集成 |
| **LiveCharts** + **LiveCharts.Wpf** | 0.9.7 | 监控页实时图表:工位时序、检测数据采集曲线(`Views/Pages/` 下 `DataCollectionPage`、`ChartModels`、`WorkflowMonitorPage`) |

UI 主入口是 `MainWindow.xaml` 的 `NavigationView`,下挂工位监控 / 告警日志 / 音频日志 / 数据采集等 Page。WinForms 与 WPF 混用,主要为托盘与少量原生交互。

## ③ 关键 NuGet 依赖

| 包 | 版本 | 用途 | 主要使用点 |
|---|---|---|---|
| **RestSharp** | 112.1.0 | HTTP 客户端,调录音盒子 REST 接口 | `Server/SmartAudio/SmartAudioClient.cs`(`RestClient`) |
| **Newtonsoft.Json** | 13.0.3 | JSON 序列化 / 反序列化(算法请求体、配置、DTO) | 全项目 `Server/`、`Models/`、`Helpers/` |
| **log4net** | 3.1.0 | 结构化文件日志,多 appender 按组件分流 | `App.config` 配置 + `Helpers/LogHelper.cs` |
| **MySql.Data** | 8.2.0 | MySQL 直连(认证、元数据、结果落库) | `Helpers/MySqlHelper.cs`(`MySqlConnection`)、`Server/SmartQuality/` |
| **Minio** | 6.0.3 | S3 协议,录音文件上传对象存储 | `Server/Storage/MinIOFileClient.cs`(`IMinioClient`) |
| **CSCore** | 1.2.1.2 | 音频解码与播放 / 混音(回放检测音频) | `Helpers/AudioPlayer.cs`(`MixingSampleProvider`) |

> 注意算法引擎调用**不用 RestSharp**:`Server/Algorithm/AlgorithmClient.cs` 用 .NET 内置 `HttpClient`(JSON + `X-API-Key`)。RestSharp 仅用于录音盒子那条 REST 链路。两套 HTTP 客户端各管一摊,见 [06-integrations.md](./06-integrations.md)。

> **log4net 分流**:`App.config` 为 `StableWorkflow`、`StableWorkChildflow`、`SmartAudioClient`、霍尔信号(含 raw CSV)等分别配了独立 RollingFileAppender,按日期滚动,落到 `Log/<组件>/`。这是现场性能诊断的主要证据来源(配套 skill `smartplc-perf`)。

## ④ 工业通信库

PLC 通信是 SmartPLC 的核心,**三种协议统一构建在私有 SDK `HslCommunication.dll` 上**(以 `Reference` 引入,非 NuGet,随构建复制到输出目录):

| 协议(对外口径) | HslCommunication 驱动类 | 代码位置 |
|---|---|---|
| **Siemens S7-1500** | `SiemensS7Net(SiemensPLCS.S1500)` | `PlcAdapter/Siemens/Siemens1500.cs` |
| **Modbus TCP** | `ModbusTcpNet`(支持 ABCD/BADC/CDAB/DCBA 字节序) | `PlcAdapter/Modbus/ModbusTCP.cs` |
| **欧姆龙** | `OmronFinsNet`(OMRON FINS TCP,CJ/CS/CP 系列) | `PlcAdapter/Omron/OmronCip.cs` |

要点:

- **三协议同源**:Modbus、欧姆龙并非独立内部实现,而是与 Siemens 一样调 HslCommunication 的对应驱动。统一 SDK 降低了多协议维护成本。协议选择 / 连接池 / 重连由 `PlcAdapter/ProtocolFactory.cs`、`ProtocolConnectionManager.cs` 收口,见 [04-plc-protocols.md](./04-plc-protocols.md)。
- **欧姆龙实际是 FINS TCP**:文件 / 类名沿用了 `OmronCip`,但底层走的是 `OmronFinsNet`(FINS 协议),日志里也明确标 "OMRON FINS TCP"。对外仍按"欧姆龙"口径描述。
- **霍尔信号**经 Modbus 链路读取(`Server/ModbusHallReader.cs`),用于精确同步产品位置,见 [03-key-concepts.md](./03-key-concepts.md)。
- **遗留依赖**:csproj 还列了 NuGet 包 `S7netplus 0.20.0`,但代码中**没有任何引用**(Siemens 走 HslCommunication)。属可清理的历史残留,不影响功能。

## ⑤ 项目结构概览

`BestPLC/` 下的主要目录(单工程,无多项目拆分):

| 目录 | 职责 |
|---|---|
| `PlcAdapter/` | 三协议适配层 + 工厂 / 连接管理 / 点位模型 |
| `Server/` | 工作流引擎(`StableWorkflow` / `StableWorkChildflow`)+ 对外集成(`SmartAudio` / `Algorithm` / `SmartQuality` / `Storage`)|
| `Services/` | 横切服务:告警日志、GC 监控、数据库调用追踪、心跳记录 |
| `Views/` | WPF 界面:`Pages/`(各功能页)、`Windows/`(弹窗)|
| `Models/` | 配置与业务数据结构(`PlcConfig`、`DetectPoint`、动态点位等)|
| `Helpers/` | 工具类:`MySqlHelper`、`LogHelper`、`AudioPlayer`、`DatabaseHelper` 等 |
| `Converters/` | WPF 值转换器 |
| `Database/` | 数据访问相关 |
| `SDK/` | 私有 SDK,含 `HslCommunication.dll` |
| `Assets/` | 图标、logo 等资源 |
| `Tools/` | 看门狗脚本(`*.bat` / `WatchdogMonitor.ps1`),随构建复制 |
| `Properties/` / `Settings.settings` | 应用设置(连接串、开关、点位配置),见 [08-deployment-config.md](./08-deployment-config.md) |

## ⑥ 延伸阅读

- 三协议适配怎么做的 → [04-plc-protocols.md](./04-plc-protocols.md)
- 这些依赖在数据流里怎么串起来 → [02-architecture.md](./02-architecture.md)
- 对外集成的地址 / 开关 / 数据格式 → [06-integrations.md](./06-integrations.md)
- 配置项怎么填 → [08-deployment-config.md](./08-deployment-config.md)
- 名词不熟 → [glossary/terms.md](../glossary/terms.md)

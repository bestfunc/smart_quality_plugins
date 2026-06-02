# 08 · 部署形态与配置

> 事实来源:`BestPLC/BestPLC.csproj`(OutputType / 依赖 / watchdog 工具)、`BestPLC/Settings.settings` + `Settings.Designer.cs`(配置项与默认值)、`BestPLC/App.config`(log4net + 连接默认)、`BestPLC/App.xaml.cs`(Mutex、线程池)。

## 部署形态

SmartPLC 是**产线现场单机部署**:一个 WPF/.NET 8 桌面应用,装在产线工控机(Windows)上,承载全部逻辑。一台中控连本线的 PLC、录音盒子、霍尔传感器,以及后端的 MySQL / MinIO / 算法服务。架构全景见 [01-product-overview.md](./01-product-overview.md) 的部署图。

| 维度 | 形态 |
|---|---|
| **载体** | 单一 WPF 应用,无容器化(无 Docker)、无独立后端服务进程 |
| **运行环境** | Windows 工控机(`net8.0-windows`,`PlatformTarget=AnyCPU`) |
| **产物** | `dotnet build` / `dotnet publish` → `WinExe` 可执行程序 + 依赖 DLL |
| **进程模型** | 单进程多线程,`async/await` + 线程池支撑多工位并发(见 [02-architecture.md](./02-architecture.md)) |
| **一机一线** | 一台中控对应一条产线,多条产线各自一台工控机独立部署 |

> 出品信息编译进程序集:`Company=巅峰表现`、`Product=SmartPLC`、`Authors=Bestfunc`(见 csproj)。

## 配置管理:Settings,编译进 exe

配置走 .NET 的 `Settings` 机制(`BestPLC/Settings.settings` + 生成的 `Settings.Designer.cs`),**默认值在编译期写入程序集**,运行时通过 `Settings.Default.XXX` 读取。

- **Application 作用域**(只读,随版本固定):`LogLevel`、`Username`、`Password`、`ENV`、`ProductID`。
- **User 作用域**(可改,运行时落到本机用户配置):绝大多数现场参数,如 MySQL / MinIO / 算法 / 阶段控制等。
- 现场调整方式:改完重新构建,或修改工控机上随 exe 的 `.config`(`App.config` 内的 `applicationSettings` / `userSettings` 段)。

> 注意:`App.config` 里另有 `ApiBaseUrl`、`ProtectionPassword`、`EnablePasswordProtection`、`StreamRequireAuth` 等项(关停程序密码保护、旧 HTTP 接口基址),其值按现场填写,凡涉及地址 / 密码一律 **(留空 / 现场配置)**,不入库本文。

## 配置项分类表

按功能域分组,列关键项 + 默认值 + 说明。密码 / 密钥类一律写"(留空 / 现场配置)"。

### 应用核心
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `LogLevel` | `Info` | 全局日志级别 |
| `ENV` | `prod` | 运行环境标识 |
| `Username` / `Password` | `admin` / `(现场配置)` | 应用内置账号(默认弱口令,现场必改) |
| `SystemUsername` | `system` | 系统操作者标识(写库审计用) |
| `OperationTimeOut` | `30` | 操作超时(秒) |

### 工位 / 产品
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `SN` | `01` | 设备序列号 |
| `StationNumber` | `01` | 工位号 |
| `ProductID` / `DefaultProductID` | `829e8326020d4319b9e7309168cd8460` | 当前 / 默认产品 ID(productId,见 [03-key-concepts.md](./03-key-concepts.md)) |
| `DefaultProductNumber` | (空) | 默认产品编号 |
| `EnableProductValidation` | `False` | 是否校验上料产品与配置一致 |

### PLC 通信
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `PollingInterval` | `50` | PLC 心跳 / 信号轮询周期(毫秒) |
| `ModbusByteOrder` | `CDAB` | Modbus 多字节序(协议细节见 [04-plc-protocols.md](./04-plc-protocols.md)) |

### 检测 / 算法
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `EnableAlgorithm` | `True` | 是否启用算法判定 |
| `EnableMicrophone` | `True` | 是否启用录音盒子采音 |
| `EnableMicrophoneV3Interface` | `False` | 是否走录音盒子 V3 接口 |
| `AlgorithmExecutePath` | `/api/v2/detect/execute` | 算法引擎执行端点 |
| `AlgorithmTimeoutSeconds` | `180` | 算法 HTTP 超时(秒) |
| `WaitAlgorithmResultTime` | `5` | 等待算法结果时长(秒) |
| `AlgorithmApiKey` | (留空 / 现场配置) | 算法鉴权 `X-API-Key` |
| `AlgorithmMode` | `0` | 算法调用模式 |
| `AlgorithmFlowNumber` / `AlgorithmVersionCode` | (空) | 算法流程号 / 版本码 |
| `EnableDirectAlgorithmCall` | `False` | 是否直连算法(绕过中间环节) |
| `AlgorithmResultAlwaysOk` | `False` | 调试用:强制判 OK(现场禁开) |
| `EnableSensitivitySync` | `False` | 是否按产品下发灵敏度给盒子(见 [03-key-concepts.md](./03-key-concepts.md)) |

### 数据库(MySQL)
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `MySqlHost` | `localhost` | MySQL 地址 |
| `MySqlPort` | `3306` | 端口 |
| `MySqlDatabase` | `bestplc` | 库名 |
| `MySqlUsername` | `root` | 账号 |
| `MySqlPassword` | (留空 / 现场配置) | 密码 |

### 文件存储(MinIO)
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `MinIOEndpoint` | (留空 / 现场配置) | S3 端点 |
| `MinIOAccessKey` / `MinIOSecretKey` | (留空 / 现场配置) | 访问密钥对 |
| `MinIOBucket` | `audio` | 音频桶名 |
| `MinIOUseSSL` | `False` | 是否启用 TLS |
| `MinIOPathEndpoint` | (留空 / 现场配置) | 路径风格端点 |

### 霍尔信号
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `EnableModbusHall` | `False` | 是否启用 Modbus 霍尔脉冲读取(霍尔信号见 [03-key-concepts.md](./03-key-concepts.md)) |
| `EnableHallControlSignal` | `False` | 是否用霍尔信号作流程控制 |

### 阶段控制
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `StageStartMode` | `Fixed1` | 阶段开始信号模式 |
| `StageReadyMode` | `Fixed1` | 阶段就绪信号模式 |
| `StageStartTimeCompensation` | `0` | 阶段开始时间补偿(秒) |
| `StageEndTimeCompensation` | `0` | 阶段结束时间补偿(秒) |
| `EnableStageReadyAutoOff` | `False` | 就绪信号是否自动复位 |
| `StageReadyAutoOffDelay` | `100` | 就绪自动复位延迟(毫秒) |
| `HeartbeatInterval` | `300` | 心跳写入间隔(毫秒) |
| `EnableNoSignalRetest` | `False` | 无信号时是否允许重测 |

### 特殊功能(自动测试 / 调试)
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `AutoTest` | `False` | 自动测试模式(无产线自检用) |
| `AutoTestDuration` | `10` | 单次自动测试时长(秒) |
| `AutoTestStageCount` | `1` | 自动测试阶段数 |
| `AutoTestInterval` | `0` | 自动测试间隔(秒) |

### 日志开关
| 配置项 | 默认值 | 说明 |
|---|---|---|
| `EnableInfoLog` / `EnableWarnLog` / `EnableErrorLog` | `True` | 各级别日志开关 |
| `EnableDebugLog` | `False` | 调试日志开关 |

> log4net 输出按组件分目录(`App.config`):`Log\StableWorkflow\`、`Log\StableWorkChildflow\`、`Log\HallSignal\`(含 Raw CSV)、`Log\General\`,按日滚动(`yyyyMMdd.log`)。

## 外部服务连接清单

现场上线前要确认这几条对外连接(地址 / 开关明细见 [06-integrations.md](./06-integrations.md)):

| 服务 | 协议 | 关键配置项 | 默认 |
|---|---|---|---|
| **MySQL**(SmartQuality 直连) | SQL/TCP | `MySqlHost/Port/Database/Username/Password` | `localhost:3306` / `bestplc` |
| **产线 PLC** | S7 / Modbus / 欧姆龙 FINS TCP | 点位表 + 设备地址(库内配置) | 按产线 |
| **Modbus(霍尔)** | Modbus TCP | `EnableModbusHall` + `ModbusByteOrder` | 关 / `CDAB` |
| **MinIO** | S3(HTTP/HTTPS) | `MinIOEndpoint/AccessKey/SecretKey/Bucket/UseSSL` | bucket `audio` |
| **算法引擎** | HTTP/JSON + X-API-Key | `AlgorithmExecutePath` + `AlgorithmApiKey` + `AlgorithmTimeoutSeconds` | `/api/v2/detect/execute` / 180s |
| **SmartAudio 录音盒子** | HTTP + WebSocket | `EnableMicrophone` + 盒子 IP/通道(库内配置) | 启用 |

## 运行保障

| 机制 | 实现 | 作用 |
|---|---|---|
| **单实例互斥** | `App.xaml.cs`:`new Mutex(true, "BesfPLC", out isNewInstance)` | 同一时刻只允许一个程序实例,重复启动弹框并退出 |
| **线程池预热** | `ThreadPool.SetMinThreads(50, 50)` | 抬高最小线程数,避免线程池缓慢增长拖慢异步任务起调延迟 |
| **服务端 GC** | csproj `ServerGarbageCollection=true` | 多核服务端 GC,降低高频检测下的停顿(诊断见 `smartplc-perf` skill) |
| **GC / 告警监控** | 启动即拉起 `GcMonitorService` + `AlertLogService` | 记录 GC 暂停、拦截 Warn/Error 告警,供现场性能诊断 |
| **看门狗工具** | `Tools\` 下随 exe 拷贝的 `*.bat` / `WatchdogMonitor.ps1` | 进程守护脚本,辅助异常退出后拉起(随构建复制到输出目录) |

> Mutex 名称在代码中拼作 `"BesfPLC"`(原样,非 BestPLC),改名前需保持一致,否则单实例保护失效。

## 延伸阅读

- 对外地址 / 开关 / 数据格式 → [06-integrations.md](./06-integrations.md)
- PLC 协议选型与点位配置 → [04-plc-protocols.md](./04-plc-protocols.md)
- 技术栈与关键依赖 → [07-tech-stack.md](./07-tech-stack.md)
- 名词不熟 → [glossary/terms.md](../glossary/terms.md)

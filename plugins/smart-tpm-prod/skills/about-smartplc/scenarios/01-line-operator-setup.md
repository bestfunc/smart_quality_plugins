# 场景 01 · 把一条已装好硬件的产线接入 SmartPLC 并跑起来

> 事实来源:本场景的事实只来自已写的 reference —— [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)、[../reference/06-integrations.md](../reference/06-integrations.md)、[../reference/08-deployment-config.md](../reference/08-deployment-config.md)。配置项默认值以 08 为准,不在此重复发明数字。

## ① 场景设定

| 项 | 内容 |
|---|---|
| **谁** | 产线运维(熟悉本线 PLC、设备布局,但不一定写代码) |
| **起点** | 某产线的硬件已装好并通电:PLC、录音盒子、霍尔传感器都在位且能 ping 通;后端 MySQL / MinIO / 算法引擎已就绪 |
| **目标** | 在中控工控机上把 SmartPLC 装起来、配好工位与点位、试跑一个产品走通"读 PLC → 录音 → 算法 → 写回"全链路 |
| **不在范围** | 现场首次对接陌生客户 PLC、按客户点位表逐点配点的实战 → 见 [./03-field-implementation.md](./03-field-implementation.md) |

> SmartPLC 是**一机一线**:一台中控对应一条产线。本场景只覆盖这一台。部署形态见 [../reference/08-deployment-config.md](../reference/08-deployment-config.md)。

## ② 上线步骤

### 步骤 1 · 装程序 + 配 MySQL

1. 把 `dotnet publish` 产物(`WinExe` + 依赖 DLL)拷到中控工控机(`net8.0-windows`,Windows)。
2. 配 MySQL 连接 —— SmartQuality 走 **MySQL 直连**(无 HTTP 开关,始终启用):`MySqlHost` / `MySqlPort`(默认 `3306`)/ `MySqlDatabase`(默认 `bestplc`)/ `MySqlUsername` / `MySqlPassword`。
3. 改默认弱口令:`Username` / `Password`(应用内置账号默认 `admin`,现场必改)。
4. 启动一次,确认能连上库(工位、检测点等配置都从库里读)。

> 配置走 .NET `Settings` 机制,默认值编译进 exe;现场调整改随 exe 的 `.config` 或重新构建。细节见 [../reference/08-deployment-config.md](../reference/08-deployment-config.md)。

### 步骤 2 · 配工位标识(SN / StationNumber)

1. 设 `SN`(设备序列号,默认 `01`)与 `StationNumber`(工位号,默认 `01`),让这台中控对上库里的检测工位(detect_station)。
2. 如要校验上料产品与配置一致,开 `EnableProductValidation`(默认 `False`);设 `ProductID` / `DefaultProductID` / `DefaultProductNumber`。

### 步骤 3 · 配 PLC 协议 / IP / 点位

1. 选协议 —— `protocol` 取值决定走哪条底层客户端,三选一:

   | 协议 | `protocol` 取值 | 默认端口 | 特有参数 |
   |---|---|---|---|
   | Siemens S7-1500 | `siemens` / `s7` | 102 | `rack`、`slot` |
   | Modbus TCP | `modbus` / `modbustcp` | 502 | `station`、`byteOrder` |
   | 欧姆龙 FINS TCP | `omron` / `omronfins` / `fins` / `finstcp` | 9600 | `sa1`、`da1` |

   > 不写 `protocol` 字段时默认按 Siemens 处理,老配置无需改动。协议对比见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)。
2. 填 PLC 设备地址与点位表(静态点位:心跳、信号、结果回写位)。
3. Modbus / 欧姆龙要确认字节序 `ModbusByteOrder`(默认 `CDAB`)。读数离谱(整数离谱、浮点 NaN)优先怀疑字节序配错。
4. 如有需要运行时写回 PLC 的值(检测值、阶段结果),配动态点位;需要缩放/平移时填 `valueTransform`(如 `*100` 把 `4.85` 写成 `485`)。
5. 按需调 `PollingInterval`(PLC 轮询周期,默认 `50` 毫秒)。

### 步骤 4 · 配智能设备(录音盒子)IP / 通道 + 检测点灵敏度

1. 在库里为本工位配智能设备(smart_device):每个录音盒子一个 `IP` + `Port` + `Token` + `Channels`(声道 1-4)。
2. 配检测点(detect_point):每个检测点绑某盒子的某个通道,并设**灵敏度**(`smallint`,范围 1-255,默认 220)。
3. 录音总开关 `EnableMicrophone`(默认 `True`;关则录音调用空跑返回成功)。
4. 如走 V3 录音接口(stop 改 `/api/v1/upload_audio` + 时间同步),开 `EnableMicrophoneV3Interface`(默认 `False`)。
5. 如要按产品向盒子下发灵敏度,开 `EnableSensitivitySync`(默认 `False`)。

> 智能设备 / 通道 / 检测点 / 灵敏度的概念见 [../reference/06-integrations.md](../reference/06-integrations.md) §1 与 [../glossary/terms.md](../glossary/terms.md)。

### 步骤 5 · 配算法 endpoint / key / flow

1. 端点 base 指向算法引擎;执行路径 `AlgorithmExecutePath`(默认 `/api/v2/detect/execute`)。
2. 鉴权 `AlgorithmApiKey`(Header `X-API-Key`,形如 `stpm-{env}-{hex}`,留空 / 现场配置)。
3. 流程号 `AlgorithmFlowNumber`(请求体必填)、`AlgorithmVersionCode`(不传走最新启用版本)。
4. 算法总开关 `EnableAlgorithm`(默认 `True`);超时 `AlgorithmTimeoutSeconds`(默认 180 秒)。
5. 如走算法直调链路(`stop_and_fetch` → MinIO → 算法自取),开 `EnableDirectAlgorithmCall`(默认 `False`),并配 MinIO:`MinIOEndpoint` / `MinIOAccessKey` / `MinIOSecretKey` / `MinIOBucket`(默认 `audio`)/ `MinIOUseSSL`。

### 步骤 6 · 试跑一个产品,验证全链路

1. 上一个产品到工位,触发检测。
2. 看主流程(StableWorkflow)是否周期读到 PLC 心跳/信号,并起子流程。
3. 看录音盒子是否按阶段采到音、算法是否返回 OK/NG、结果是否写回 PLC。
4. 看库里 `detect` / `detect_stage` / `detect_channel` 是否落库;算法直调时 MinIO 里有无音频文件。
5. 全部对上即上线成功。链路时序见 [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md)。

> 无产线想空跑自检:开 `AutoTest`(默认 `False`)配合 `AutoTestDuration` / `AutoTestStageCount` / `AutoTestInterval`。调试用的 `AlgorithmResultAlwaysOk`(强制判 OK)现场禁开。

## ③ 上线前检查清单

| # | 检查项 | 配置 / 依据 | 通过标准 |
|---|---|---|---|
| 1 | 程序已装、能启动 | `dotnet publish` 产物 + Windows 工控机 | 单实例正常起(Mutex `BesfPLC`) |
| 2 | MySQL 连得上 | `MySqlHost/Port/Database/Username/Password` | 工位 / 检测点配置能读出 |
| 3 | 弱口令已改 | `Username` / `Password` | 不再是默认 `admin` |
| 4 | 工位标识对上库 | `SN` / `StationNumber` | 对应到正确的 detect_station |
| 5 | PLC 协议 / IP / 端口正确 | `protocol` + 设备地址 | PLC 连接建立、心跳在跳 |
| 6 | 字节序正确(Modbus/欧姆龙) | `ModbusByteOrder`(默认 `CDAB`) | 读数不离谱、无 NaN |
| 7 | 录音盒子 IP / 通道 / token 对 | 库内 smart_device | start/stop 通,采到音 |
| 8 | 检测点灵敏度已设 | `detect_point.sensitivity`(1-255) | 每个检测点都有值 |
| 9 | 算法 endpoint / key / flow 配齐 | `AlgorithmExecutePath` / `AlgorithmApiKey` / `AlgorithmFlowNumber` | 试跑能返回 OK/NG |
| 10 | (直调时)MinIO 可写 | `MinIOEndpoint/AccessKey/SecretKey/Bucket` | 音频文件能落桶 |
| 11 | 调试开关已关 | `AlgorithmResultAlwaysOk` / `AutoTest` | 均为 `False` |
| 12 | 试跑一个产品全链路通 | 步骤 6 | 库三表落库 + 结果写回 PLC |

## ④ 常见卡点

| 现象 | 可能原因 | 处理方向 |
|---|---|---|
| 读 PLC 数值离谱 / 浮点 NaN | Modbus / 欧姆龙字节序配错 | 调 `ModbusByteOrder`(四档:`ABCD`/`BADC`/`CDAB`/`DCBA`),见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md) |
| PLC 连不上 | `protocol` / IP / 端口 / 特有参数错(S7 缺 `rack`/`slot`,欧姆龙缺 `sa1`/`da1`) | 按协议表逐项核对;连接 30 秒会自动重连一次 |
| 写回 PLC 整数掉精度 | 期望整数寄存器但值带小数 | 用 `valueTransform`(如 `*100`)转定点;整数写入按四舍五入取整 |
| 录音"成功"但其实没采到 | `EnableMicrophone` 关了(空跑返回成功) | 确认开关为 `True` |
| 盒子连不上 / 鉴权失败 | IP / 端口 / token 配错(Header `X-Token`) | 核对库内 smart_device 的 IP / Port / Token |
| 算法返回非成功 | endpoint / `X-API-Key` / `AlgorithmFlowNumber` 缺或错 | 成功需 HTTP 200 且 body `code==200`;核对三项 |
| 算法直调拿不到文件 | 没开 `EnableDirectAlgorithmCall` 或 MinIO 配置缺 | 开直调开关并配齐 MinIO,`file[].path` 给算法自取 |
| 结果对不上工位 | `SN` / `StationNumber` 没对上库里工位 | 核对工位标识 |
| 程序起不来 / 重复弹框 | 已有实例在跑(单实例互斥 Mutex `BesfPLC`) | 关掉旧实例再起 |

## ⑤ 延伸阅读

- 配置项全清单 + 默认值 → [../reference/08-deployment-config.md](../reference/08-deployment-config.md)
- PLC 协议选型 / 点位模型 / 连接重连 → [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)
- 对接盒子 / 算法 / MySQL / MinIO 的地址与开关 → [../reference/06-integrations.md](../reference/06-integrations.md)
- 一个产品被检测的完整时序 → [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md)
- 现场对接陌生客户 PLC 的实战 → [./03-field-implementation.md](./03-field-implementation.md)
- 名词不熟(检测工位 / 通道 / 灵敏度 / dataContent) → [../glossary/terms.md](../glossary/terms.md)

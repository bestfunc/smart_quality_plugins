# 06 · 对外集成

> 事实来源:`BestPLC/Server/SmartAudio/SmartAudioClient.cs`(+ `.DirectFetch.cs`、`SmartAudioClientStreamer.cs`)、`Server/Algorithm/AlgorithmClient.cs` + `AlgorithmRequest.cs`/`AlgorithmResponse.cs`、`Server/SmartQuality/SmartQualityClientV2.cs`、`Server/Storage/MinIOFileClient.cs`、`Settings.settings`、`Docs/盒子stop_and_fetch接口规范.html`。

## 概览

SmartPLC 作为产线中控,除了 PLC(见 [04-plc-protocols.md](./04-plc-protocols.md))外,还对接 4 个后端外部系统。每个集成都有独立入口类和 `Settings` 里的开关。

| 系统 | 协议 | 入口类 | 触发时机 | 主开关 |
|---|---|---|---|---|
| **SmartAudio 录音盒子** | HTTP/1.1(REST)+ WebSocket | `SmartAudioClient` / `SmartAudioClientStreamer` | 阶段录音、实时听音 | `EnableMicrophone` |
| **算法引擎** | HTTP/JSON + `X-API-Key` | `AlgorithmClient` | 拿到音频后判 OK/NG | `EnableAlgorithm` |
| **SmartQuality(MySQL)** | MySQL 直连 + 事务 | `SmartQualityClientV2` | 元数据读取、结果落库 | (无开关,始终启用) |
| **MinIO** | S3 兼容(HTTP/HTTPS) | `MinIOFileClient` | 录音持久化 | 走算法直调链路时启用 |

> 录音盒子 / 算法引擎都有"V3 直调"新链路与旧链路并存,由 `EnableDirectAlgorithmCall` 等开关切换,见 §6。

---

## ① SmartAudio 录音盒子

录音盒子是挂在工位下的智能设备(`smart_device`,一个 IP + 多声道)。一个工位可挂多个盒子,SmartPLC 用 `MicrophoneConfig`(`IP` / `Port` / `Token` / `Channels`)为每个盒子建一个 `SmartAudioClient`。

**鉴权与连接**:REST 客户端基址 `http://{IP}:{Port}`,默认 Header `X-Token: {token}`(token 从 `smart_device.token` 读),普通 start/stop 超时 3 秒。

**两类通道**:

| 用途 | 入口 | 协议 | 端点 |
|---|---|---|---|
| 阶段录音(检测链路) | `SmartAudioClient.Start` / `Stop` | REST | `/api/v1/start`、`/api/v1/stop`(V3 模式 stop 改 `/api/v1/upload_audio`) |
| 算法直调拉流 | `SmartAudioClient.StopAndFetchAudio` | REST | `/api/v1/stop_and_fetch` |
| 实时听音(UI 波形/播放) | `SmartAudioClientStreamer` | WebSocket | `ws://{IP}:{Port}/api/v1/stream/live` |
| 时间同步 | `SmartAudioClient.PushTimeSync` | REST | `/api/v1/time_sync` |

**时间同步**:V3 接口模式下,`Start` 前会先推时间同步(client/server 用 Unix 纳秒时间戳),按目标 IP 限流——同一盒子 5 分钟内只同步一次(`TimeSyncInterval`)。同步失败只记 Warn,不阻断录音。

**stop_and_fetch(算法直调)**:接口名是历史命名,实际语义是**按时间窗截取片段**(`startTime`/`endTime` 为 Unix 毫秒),录音 daemon 不停录,以便同一 workflow 连续 fetch 多个 stage。请求体含 `external`(透传算法)/ `channels` / `detect{detectStageID,startTime,endTime}` / `return_audio:true`;响应单通道用 `application/octet-stream` + 自定义 Header(`X-File-Md5` / `X-Duration` / `X-File-Name`),多通道用 `multipart/form-data`(每 part 带 `channel=N`)。该客户端超时放宽到 **3 分钟**。完整规范见 `Docs/盒子stop_and_fetch接口规范.html`。

> WebSocket 流固定 44.1kHz / 16bit,订阅通道范围 1-4,5 秒无数据视为断连。它只服务 UI 听音,不参与 OK/NG 判定。

---

## ② 算法引擎

新算法引擎客户端 `AlgorithmClient` 走 **`POST /api/v2/detect/execute`**,`application/json` + `X-API-Key` Header(旧 multipart 协议已废弃)。

| 项 | 值 | 来源 |
|---|---|---|
| 端点 | base(`AlgorithmRequest.Endpoint`)+ `/api/v2/detect/execute` | `AlgorithmExecutePath` |
| 鉴权 | Header `X-API-Key`(形如 `stpm-{env}-{hex}`) | `AlgorithmApiKey` |
| 超时 | **180000 ms(180 秒)** 默认 | `AlgorithmTimeoutSeconds` |
| 响应上限 | `HttpClient` 512 MB,启用 GZip/Deflate 解压 | `AlgorithmClient.cs` |

**请求体字段**(严格对齐对接系统 `_call_detect_execute`):`dataContent`(核心,见下)、`mode`(`"0"`=实际 / `"1"`=全 OK / `"2"`=全 NG,仅调试)、`device_type` / `device_id`(生产传空串);`flowNumber`(必填,`AlgorithmFlowNumber`)、`versionCode`(不传走最新启用版本)、`disabledRules`(测试专属)仅在有值时才放进 body。

**dataContent**:嵌套数据结构,含 `detect[]` / `detect_channel[]` / `detect_point[]` / `detect_station_stage[]` / `file[]`(及可选 `testRecordNumber`)。文件不再随请求上传——`file[].path` 给 MinIO URL,算法侧自己 GET。

**响应解析**:HTTP 200 **且** body `code==200` 才算成功;结果取 `data.point_first` 视图(`productResult` + `pointResults[].stageResults[]`)。结果映射:`positive→OK` / `negative→NG` / `OK→OK` / `NG→NG` / 其他→null。`stageKey` 形如 `{pointKey}__stage{N}`,调用方按 (PointKey, StageIndex) 路由回对应 `detect_channel`。

---

## ③ SmartQuality(MySQL 直连)

`SmartQualityClientV2` **直连 MySQL**(经 `DatabaseHelper`),不走 HTTP API——换取多阶段结果的**事务一致性**与更低延迟(取舍背景见 [02-architecture.md](./02-architecture.md))。连接参数来自 `Settings`:`MySqlHost` / `MySqlPort` / `MySqlDatabase`(默认 `bestplc`)/ `MySqlUsername` / `MySqlPassword`。

**事务**:创建检测阶段、建产品等"读校验 + 多表写入"用 `_dbHelper.BeginTransaction()`,失败 `Rollback`、成功 `Commit`(如 `CreateDetectStage` 一次性写 `detect_stage` + 多条 `detect_channel`)。每个方法调用经 `RecordMethodCallAsync` 把 SQL 操作留痕到数据库调用记录(可观测兜底)。

**核心表**(节选):

| 表 | 用途 | 典型方法 |
|---|---|---|
| `detect_station` | 检测工位 + PLC 绑定 | `GetDetectStations` / `GetDetectStationInfo` |
| `detect_point` | 检测点(绑通道、带灵敏度) | `GetDetectStationInfo` |
| `smart_device` | 录音盒子(IP / token / 通道) | `GetDetectStationInfo` |
| `detect` | 一次检测主记录(产品级) | `CreateDetectMain` / `SaveAlgorithmMainResult` |
| `detect_stage` | 阶段记录(1-8) | `CreateDetectStage` / `SaveAlgorithmStageResult` |
| `detect_channel` | 通道级结果(`aiResult` / `fileID`) | `GetChannelDetectResult` / `SaveAlgorithmStageResult` |
| `product` | 产品型号 | `GetProductIdByNumber` / `CreateProduct` |

> 工位/检测点/通道/阶段等概念定义见 [03-key-concepts.md](./03-key-concepts.md)。

---

## ④ MinIO

`MinIOFileClient` 用 S3 兼容 SDK(`Minio` NuGet)持久化录音文件。配置来自 `Settings`:`MinIOEndpoint` / `MinIOAccessKey` / `MinIOSecretKey` / `MinIOUseSSL`,**bucket 默认 `audio`**(`MinIOBucket` 为空时回落到 `"audio"`)。

- **写入**(`PutObjectAsync`):上传前 `EnsureBucketAsync` 检查并按需建桶;默认 `contentType=audio/wav`;返回 `Bucket` / `ObjectKey` / `Size` / `ETag`。
- **读取**(`GetObjectAsync`):拉到内存流返回。
- **写桶失败容错**:`BucketExists` 检查异常时只记 Warn 并继续尝试 put,不直接中断主流程。

在算法直调链路里,SmartPLC `stop_and_fetch` 拿到字节流后写 MinIO,再把 `file[].path`(MinIO URL)交给算法侧自取,完成"盒子 → SmartPLC → MinIO → 算法"闭环。

---

## ⑤ 集成开关(Settings.Enable*)

`Settings.settings` 里与集成相关的开关一览(默认值以代码为准):

| 开关 | 默认 | 作用 |
|---|---|---|
| `EnableMicrophone` | `True` | 总开关:关则录音调用直接返回成功(空跑) |
| `EnableMicrophoneV3Interface` | `False` | 走 V3 录音接口(start 不调盒子、stop 改 `/api/v1/upload_audio`、启用时间同步) |
| `EnableAlgorithm` | `True` | 是否调用算法引擎判定 |
| `AlgorithmResultAlwaysOk` | `False` | 调试:强制算法结果全 OK |
| `EnableDirectAlgorithmCall` | `False` | 走 `stop_and_fetch` 算法直调链路(否则走旧 edge_api 上传链路) |
| `EnableSensitivitySync` | `False` | 按产品向盒子下发灵敏度 |
| `EnableModbusHall` | `False` | 启用霍尔脉冲同步(Modbus) |
| `ENV` | `prod` | `dev` 时录音 / 流连接走模拟,不打真实盒子 |

> 算法侧还有非 Enable 前缀的配置:`AlgorithmFlowNumber` / `AlgorithmVersionCode` / `AlgorithmApiKey` / `AlgorithmMode` / `AlgorithmTimeoutSeconds`。完整配置项分类见 [08-deployment-config.md](./08-deployment-config.md)。

---

## 延伸阅读

- 配置怎么填、外部服务连接清单 → [08-deployment-config.md](./08-deployment-config.md)
- 集成在检测时序里的位置 → [05-detection-workflow.md](./05-detection-workflow.md)
- 技术栈与 NuGet 依赖(RestSharp / Minio / MySqlConnector 等) → [07-tech-stack.md](./07-tech-stack.md)
- 名词不熟(dataContent / 通道 / 灵敏度) → [glossary/terms.md](../glossary/terms.md)

# FAQ · 对内开发常见问题

> 事实来源:`../reference/02-architecture.md`、`04-plc-protocols.md`、`05-detection-workflow.md`、`06-integrations.md`、`07-tech-stack.md`(均以代码为准)。本页面向上手 SmartPLC(代码库 `BestPLC`)的开发同事,解释那些"为什么这么写"的工程取舍,不替代 reference 的完整事实。

新人读这套代码最容易卡在几个"看着别扭、其实有原因"的地方。下面按困惑频次排,每条给结论 + 代码依据 + 出处。术语不熟先翻 [../glossary/terms.md](../glossary/terms.md)。

---

## 架构取舍

### Q1. 为什么质检平台是直连 MySQL,而不是走 HTTP API?

`SmartQualityClientV2` 经 `DatabaseHelper` **直连 MySQL**,不经任何后端 HTTP 服务。两个原因:

1. **事务一致性**。一次检测要跨 `detect` / `detect_stage` / `detect_channel` 多表写入,必须原子。代码里"读校验 + 多表写入"的方法都包在 `_dbHelper.BeginTransaction()` 里——失败 `Rollback`、成功 `Commit`。例如 `CreateDetectStage` 一次性写 `detect_stage` + 多条 `detect_channel`。隔着 HTTP API 很难把这种多表写做成一个事务。
2. **低延迟**。v3.5 是高频调用场景(每工位每阶段都要落库 + 写回),直连省掉一层 HTTP 序列化/网络往返开销。

代价是中控要拿到库凭据、和库的 schema 强耦合——这是有意接受的取舍。背景见 [../reference/02-architecture.md](../reference/02-architecture.md) 的"设计要点"与 [../reference/06-integrations.md](../reference/06-integrations.md) §3。

> 注意:`SmartQuality(MySQL)` 是**唯一没有 Enable 开关**的集成,始终启用。录音盒子、算法都能被开关关掉空跑,DB 不能。

### Q2. 多工位并发,工位之间怎么不互相拖累?

三个机制叠加:

| 机制 | 做法 | 出处 |
|---|---|---|
| **每工位独立主流程** | 一个工位一个常驻 `StableWorkflow`,子流程按产品创建/销毁,互不共享状态 | [05](../reference/05-detection-workflow.md) |
| **Server GC** | csproj `ServerGarbageCollection=true`,多后台堆减少停顿,多工位并发吞吐优先 | [07](../reference/07-tech-stack.md) ① |
| **线程池预热** | `App` 启动时 `ThreadPool.SetMinThreads(50,50)`,降低异步起调延迟 | [02](../reference/02-architecture.md) |

另外主流程内部把定时器拆成三个(主轮询 50ms / 心跳 300ms / 维护 1s),心跳写单独一根,就是为了"心跳别被主轮询的耗时操作拖慢"。每阶段的录音/算法/落库都丢到并行 `Task` 里,不阻塞那根 50ms 主轮询。

---

## 算法接口

### Q3. 怎么调算法引擎?为什么不用 RestSharp?

算法走 **`POST /api/v2/detect/execute`**,`AlgorithmClient` 用 **.NET 内置 `HttpClient`**,不是 RestSharp。关键点:

- **Content-Type**:`application/json`(旧 multipart 协议已废弃)。
- **鉴权**:Header `X-API-Key`(形如 `stpm-{env}-{hex}`),不是 Bearer。
- **请求体核心是 `dataContent`**:嵌套结构,含 `detect[]` / `detect_channel[]` / `detect_point[]` / `detect_station_stage[]` / `file[]`。文件不随请求上传,`file[].path` 给 MinIO URL,算法侧自己 GET。
- **判成功要两层**:HTTP 200 **且** body `code==200`,缺一不可。
- **结果路由**:取 `data.point_first` 视图,`stageKey` 形如 `{pointKey}__stage{N}`,按 (PointKey, StageIndex) 路由回对应 `detect_channel`。

**为什么是 HttpClient 而不是 RestSharp**:RestSharp(112.1.0)在本项目里**只服务录音盒子那条 REST 链路**(`SmartAudioClient`)。算法那条是后来切的 JSON + X-API-Key 协议,单独用 HttpClient,两套 HTTP 客户端各管一摊,别混用。详见 [../reference/06-integrations.md](../reference/06-integrations.md) §2 与 [../reference/07-tech-stack.md](../reference/07-tech-stack.md) ③ 的注记。

### Q4. `mode` / `device_id` / `flowNumber` 这些字段什么时候传?

| 字段 | 生产取值 | 说明 |
|---|---|---|
| `mode` | `"0"` | `"0"`=实际 / `"1"`=全 OK / `"2"`=全 NG(后两个仅调试) |
| `device_type` / `device_id` | 空串 | 生产传空 |
| `flowNumber` | `AlgorithmFlowNumber` | **必填** |
| `versionCode` | 不传 | 不传则走最新启用版本 |
| `disabledRules` | 仅有值时放进 body | 测试专属 |

默认超时 **180000ms(180 秒)**(`AlgorithmTimeoutSeconds`),响应上限 512MB 并启用 GZip/Deflate 解压。

### Q5. 算法没结果会卡死流程吗?

不会。等结果是**轮询数据库**(每 50ms 查一次),最长等 `WaitAlgorithmResultTime`(默认 5s),超时写兜底 `Code=50/timeout`,流程继续往下走,不阻塞。另外 `EnableAlgorithm=false` 时,结果落库标 `Test`、PLC 仍写默认 `Code=100`,算法流程在后台照走。见 [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md) 的"异常/超时/重置处理"。

---

## 点位写入的坑

### Q6. 动态点位写整型寄存器,为什么用 `Math.Round` + `AwayFromZero`?

因为**直接强转会截断,吃掉精度**。

动态点位带可选的值变换 `valueTransform`(如 `*100` 把小数转定点整数)。写 `Int`/`Short` 型动态点时,变换结果用:

```
Math.Round(value, MidpointRounding.AwayFromZero)
```

取整后再转目标类型。`Float` 型则直接 `(float)` 转换,不取整。

**为什么不能直接 `(int)`**:浮点运算有误差。代码注释举的例子是 `4.85 * 100 = 484.999...`,直接强转截断会变成 `484`,丢了 `0.01`——本该写 `485`。`AwayFromZero` 是远离零取整,保证 `.5` 这种边界也朝绝对值大的方向进,不会被吞。任务描述里 `2.599*100=259.9` 同理:截断成 259 就差了一个最小刻度。

值变换四种(首字符是运算符):`*N` 乘 / `/N` 除(除数 0 跳过)/ `+N` 加 / `-N` 减;表达式空或非法时原值返回。实现在 `StableWorkChildflow.ApplyValueTransform`。详见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md) 的"动态点位的值变换"。

### Q7. 写回 PLC 的结果值约定是什么?

| 值 | 含义 |
|---|---|
| `-1` | 未就绪 / 复位 |
| `1` | OK |
| `2` | NG |

阶段结果写 `WPointStage{N}Result`(stage 1..8 的 `switch` 映射写死在 `WriteStageResult`),综合结果写 `WPointResult` + `WPointErrorCode`,再写 `WPointResultNotify=1` 通知 PLC 读取。`AlgorithmResultAlwaysOk=true` 时落库保留真实结果,但 `WPointResult` 强制写 1(产线放行调试用)。见 [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md) 的"结果聚合"。

---

## 协议适配

### Q8. 为什么上层只认 `IPlc<SiemensPlcPoint>`,还要套一层适配器?

历史原因。主工作流 `StableWorkflow` 历史上只认 `IPlc<SiemensPlcPoint>`。为了不动上层,非西门子协议时用 `MultiProtocolAdapter` 包一层:运行时 `ConvertToGenericPoint` 把西门子点位转成目标协议点位(`ModbusTCPPoint` / `OmronCipPoint`)。

- `protocol` 是 Siemens 时**不套适配器**,直接用 `Siemens1500`,少一层转换开销。
- 配置里没写 `protocol` 字段时**默认按 Siemens 处理**(`ProtocolFactory` 与 `MultiProtocolAdapter` 都默认 Siemens),老配置无需改。

三协议底层**统一压在私有 SDK `HslCommunication.dll`** 上(`SiemensS7Net` / `ModbusTcpNet` / `OmronFinsNet`),Modbus、欧姆龙不是独立内部实现。见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md) 的"统一适配层"与 [../reference/07-tech-stack.md](../reference/07-tech-stack.md) ④。

### Q9. 想加第四种 PLC 协议,改哪里?

按现有四个收口点扩(均在 `PlcAdapter/`):

1. **驱动**:接入 HslCommunication 对应驱动类(若它支持),否则自己实现底层客户端。
2. **点位类型**:仿 `ModbusTCPPoint` / `OmronCipPoint` 实现 `IPlcPoint`。
3. **工厂**:在 `ProtocolFactory` 的 `protocol` 字符串 `switch` 加分支,补 `CreateConnection` / `CreatePoint` / `CreateDynamicPoint`。
4. **适配器**:仿 `ModbusPlcAdapter` / `OmronCipPlcAdapter` 把新客户端包成 `IPlc<IPlcPoint>`,`MultiProtocolAdapter` 会自动把它适配成上层要的 `IPlc<SiemensPlcPoint>`。

连接池、自动重连(`ProtocolConnectionManager` 内 30s 巡检 Timer)、超时这些公共能力不用动,新协议自动复用。注意字节序:多字节类型的字节序由 `ModbusByteOrder` 控制(`ABCD`/`BADC`/`CDAB` 默认/`DCBA`)。

> 命名提醒:欧姆龙实现类叫 `OmronCip`,但底层是 **FINS TCP**(`OmronFinsNet`),不是 EtherNet/IP CIP,`Cip` 只是历史命名。别被名字误导。

---

## 排查 / 可观测

### Q10. 现场出问题,日志怎么按组件分流找?

`log4net`(3.1.0)在 `App.config` 里给主要组件各配了独立 `RollingFileAppender`,按日期滚动,落到 `Log/<组件>/`。排查时按组件直奔对应目录:

| 现象 | 看哪个组件日志 |
|---|---|
| 心跳/信号/产品号问题 | `StableWorkflow` |
| 单产品检测、阶段、写回异常 | `StableWorkChildflow`(还有按 `_instanceId` 分的产品级目录) |
| 录音盒子启停/拉流 | `SmartAudioClient` |
| 产品位置同步异常 | 霍尔信号(含 raw CSV) |

子流程还会按 `_instanceId` 写独立日志目录,便于按单个产品追溯。这是现场性能诊断的主要证据来源,配套用开发类 skill `smartplc-perf`。见 [../reference/07-tech-stack.md](../reference/07-tech-stack.md) ③ 的"log4net 分流"。

### Q11. GC 监控在防什么?

防**长 GC 暂停拖垮工位实时性**。多工位各跑异步循环,主轮询是 50ms 级、心跳是 300ms 级——一次几百毫秒的 GC 停顿就可能让心跳读不到变化、被判 PLC 不健康(`IsHealthy` 要求"读心跳 5 秒内有变化")。所以:

- 编译期用 **Server GC**(多后台堆,减停顿)。
- 运行期由 `Services/GcMonitorService.cs` 持续监控 GC 暂停,作为现场性能诊断兜底。

这是 [../reference/02-architecture.md](../reference/02-architecture.md) "可观测优先"的一部分,和告警拦截、数据库调用追踪(`RecordMethodCallAsync` 把每次 SQL 操作留痕)一起,为现场根因分析提供证据链。

---

## 集成开关速查

写代码/联调前先确认这些开关状态(默认值以代码为准),很多"为什么没调盒子/没判 OK/NG"的疑问其实是开关没开:

| 开关 | 默认 | 关掉/打开会怎样 |
|---|---|---|
| `EnableMicrophone` | `True` | 关则录音调用直接返回成功(空跑) |
| `EnableMicrophoneV3Interface` | `False` | 开则走 V3 录音接口(stop 改 `/api/v1/upload_audio`、启用时间同步) |
| `EnableAlgorithm` | `True` | 关则落库标 `Test`、PLC 写默认 `Code=100` |
| `AlgorithmResultAlwaysOk` | `False` | 开则 `WPointResult` 强制 1(真实结果仍落库) |
| `EnableDirectAlgorithmCall` | `False` | 开则走 `stop_and_fetch` 算法直调链路 |
| `EnableSensitivitySync` | `False` | 开则按产品向盒子下发灵敏度 |
| `EnableModbusHall` | `False` | 开则启用霍尔脉冲同步 |
| `ENV` | `prod` | `dev` 时录音/流连接走模拟,不打真实盒子 |

完整开关与配置项见 [../reference/06-integrations.md](../reference/06-integrations.md) §5 与 [../reference/08-deployment-config.md](../reference/08-deployment-config.md)。

---

## 延伸阅读

- 全局架构地图 → [../reference/02-architecture.md](../reference/02-architecture.md)
- 检测主/子流程 8 阶段时序 → [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md)
- 协议适配与点位模型 → [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)
- 对外集成地址/开关/数据格式 → [../reference/06-integrations.md](../reference/06-integrations.md)
- 技术栈与依赖 → [../reference/07-tech-stack.md](../reference/07-tech-stack.md)
- 客户/评估者常问(对外口径) → [./01-customer-questions.md](./01-customer-questions.md)
- 名词不熟 → [../glossary/terms.md](../glossary/terms.md)

# 场景 03 · 现场实施:到客户现场对接 PLC 上线

> 事实来源(以这三篇 reference 为准):[../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)、[../reference/06-integrations.md](../reference/06-integrations.md)、[../reference/08-deployment-config.md](../reference/08-deployment-config.md)。本篇是操作场景,不发明数字,所有参数以 reference 为准。

## 场景设定

你是集成商 / 设备厂商派到客户现场的实施工程师。客户的产线**已经建好**,PLC、产线机械、上下料都是别人做的;你要做的是把 **SmartPLC 中控**插进这条线:在产线工控机上装好这个 WPF/.NET 8 单机应用,让它能读写现场 PLC、驱动录音盒子采音、把音频送算法判 OK/NG、再把结果写回 PLC 控制流转。

现场你手里通常有:产线电气工程师给的 **PLC 点位表**(地址 + 含义)、录音盒子的 **IP / 通道 / token**、后端给的 **MySQL / MinIO / 算法引擎**地址与密钥。本篇按"摸 PLC → 选协议 → 配点 → 联调 → 试运行"的顺序走一遍。

> 上一道工序(库内配工位 / 检测点 / 灵敏度)见 [./01-line-operator-setup.md](./01-line-operator-setup.md);本篇聚焦"对接现场 PLC 和外部服务"。

---

## ① 摸 PLC 品牌 → 选协议

到现场第一件事是问清产线 PLC 是什么牌子,对照下表定 `protocol` 取值。三种协议底层都走 HslCommunication,选型只是填一个配置字符串,上层工作流无感。

| 现场 PLC | `protocol` 取值(任填其一) | 默认端口 | 特有参数 |
|---|---|---|---|
| Siemens S7-1500 | `siemens` / `s7` | 102 | `rack`、`slot` |
| 第三方设备 / 仪表 / 杂牌(只给寄存器) | `modbus` / `modbustcp` | 502 | `station`、字节序 `byteOrder` |
| 欧姆龙(CJ/CS/CP 系列,FINS TCP) | `omron` / `omronfins` / `fins` / `finstcp` | 9600 | `sa1`、`da1`(FINS 节点号) |

检查清单:

1. 确认品牌 + 系列,定 `protocol` 字符串。
2. 西门子:拿到 `DB` 块号、`rack` / `slot`(常见 0/1)。
3. 欧姆龙:确认内存区寻址用的是字符串地址(`D100`/`W100`/`H100`/`A100`/`C100`/`T100`),并问清 FINS 节点号 `sa1`/`da1`。
4. Modbus:确认寄存器地址从 0 起(`AddressStartWithZero=true`),问清从站号 `station`。
5. **老配置兜底**:配置里没写 `protocol` 字段时,系统默认按 Siemens 处理 —— 对接非西门子线,务必显式填 `protocol`,别依赖默认。

> 协议命名坑:欧姆龙实现类叫 `OmronCip`,但走的是 **FINS TCP**,不是 EtherNet/IP CIP,`Cip` 只是历史命名。别被类名误导。

---

## ② 按点位表配静态点 + 动态点

点位分两类,都由系统按 `protocol` 自动造出对应协议的点位类型,你只管按点位表填地址。

| 点位类型 | 是什么 | 现场怎么配 |
|---|---|---|
| **静态点位** | 配置里固定的读写点:心跳、阶段开始 / 就绪信号、结果回写位 | 按点位表逐条填地址,工作流主链路用 |
| **动态点位** | 运行时按流程节点 / 阶段 / 通道写入 PLC 的点:检测值、阶段结果下发 | 配地址 + 可选 `valueTransform` 值变换 |

### 动态点的值变换(valueTransform)

很多 PLC 寄存器只能存整数,而检测值是小数(例如 `4.85`)。动态点位带一个可选的**单运算值变换表达式**,首字符是运算符:

| 表达式 | 含义 | 例 |
|---|---|---|
| `*N` | 乘 | `*100`:把 `4.85` 写成 `485`(小数转定点整数) |
| `/N` | 除(除数为 0 时跳过变换) | `/10` |
| `+N` / `-N` | 加 / 减(偏移) | `+100` |

整型寄存器的关键注意:

- 写 `Int` / `Short` 型动态点时,变换结果用**四舍五入**(`MidpointRounding.AwayFromZero`)再转整型,**不是截断**。这是为避免浮点误差:`4.85*100` 实际算出 `484.999...`,直接强转会丢掉 `0.01` 变成 `484`,四舍五入回到 `485`。
- `Float` 型动态点直接 `(float)` 转换,不取整。
- 表达式为空或非法时,原值返回(不变换)。

配点检查清单:

1. 静态点(心跳 / 信号 / 回写位)逐条对点位表填地址。
2. 动态点确认目标寄存器数据类型(Int/Short/Float),决定要不要 `valueTransform`、要不要取整。
3. 小数检测值要进整型寄存器 → 用 `*100` 之类缩放成定点整数,并和客户确认下游怎么解读这个缩放。
4. 动态点若指定了独立 PLC 地址,系统会走动态连接池单独建连(见 ⑤ 连接管理)。

---

## ③ 字节序与寻址注意(Modbus / 欧姆龙)

多字节类型(short / int / float)的字节序由 `ModbusByteOrder` 决定,四档,**默认 `CDAB`(字交换)**:

| 取值 | 含义 |
|---|---|
| `ABCD` | 大端 |
| `BADC` | — |
| `CDAB` | 字交换(默认值) |
| `DCBA` | 小端 |

排查口诀:**读数明显异常优先怀疑字节序配错** —— 整数离谱、浮点变 NaN,先翻 `ModbusByteOrder` 改一档试。寻址上还要确认 Modbus 寄存器地址从 0 起(`AddressStartWithZero=true`)。细节见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md) 与 `Docs/ModbusHighLowByteConfig.md`。

> 西门子 S7 走 S7 字节偏移寻址,一般不涉及这套 `ModbusByteOrder` 字节序问题;欧姆龙多字节同样受字节序影响,异常时同理排查。

---

## ④ 联调录音盒子(SmartAudio)

录音盒子是挂在工位下的智能设备(一个 IP + 多声道),库内按 `smart_device` 配 IP / token / 通道,SmartPLC 为每个盒子建一个客户端。现场要确认连得通、通道对得上、时间同步正常。

检查清单:

1. **总开关**:`EnableMicrophone` 默认 `True`;关掉时录音调用直接空跑返回成功(联调初期排查可临时关)。
2. **连接**:REST 基址 `http://{IP}:{Port}`,默认带 Header `X-Token`(token 从 `smart_device.token` 读);普通 start/stop 超时 3 秒。
3. **通道**:盒子声道号 1–4,直接对应检测点的 `channel`。确认点位表里 detect_point 的通道和盒子物理声道一一对上,别接反。
4. **接口模式**:`EnableMicrophoneV3Interface` 默认 `False`。走 V3 时 start 不调盒子、stop 改 `/api/v1/upload_audio`,并启用时间同步。
5. **时间同步**:V3 模式 `Start` 前会先推时间同步(Unix 纳秒时间戳),同一盒子 5 分钟内只同步一次;同步失败只记 Warn 不阻断录音。现场先确保工控机与盒子时钟别差太离谱。
6. **实时听音**:UI 波形 / 播放走 WebSocket(`ws://{IP}:{Port}/api/v1/stream/live`),固定 44.1kHz / 16bit,5 秒无数据视为断连。它只服务 UI 听音,**不参与 OK/NG 判定**,联调时可用它确认麦克风真的在拾音。
7. `ENV` 为 `dev` 时录音 / 流连接走模拟、不打真实盒子;现场上真机务必确认 `ENV=prod`。

---

## ⑤ 联调算法引擎

算法引擎接收音频返回 OK/NG。客户端走 `POST /api/v2/detect/execute`,`application/json` + `X-API-Key` Header。

| 项 | 现场要确认的值 | 配置项 |
|---|---|---|
| 端点 | base + `/api/v2/detect/execute` | `AlgorithmExecutePath` |
| 鉴权 | Header `X-API-Key`(形如 `stpm-{env}-{hex}`) | `AlgorithmApiKey`(留空 / 现场配置) |
| 超时 | 默认 180 秒 | `AlgorithmTimeoutSeconds` |
| 流程号 | 必填,对应算法侧建模流程 | `AlgorithmFlowNumber` |
| 版本码 | 不传走最新启用版本 | `AlgorithmVersionCode` |

检查清单:

1. **总开关**:`EnableAlgorithm` 默认 `True`。
2. 填好 `AlgorithmExecutePath` base 地址 + `AlgorithmApiKey`,确认能通到算法服务。
3. **flowNumber 必填**:这是算法侧的建模流程号,**建模导入期**要和算法 / 建模同事对齐 —— 流程号 / 版本对不上,算法找不到模型。
4. **音频闭环**:走直调链路(`EnableDirectAlgorithmCall=True`)时,盒子 `stop_and_fetch` 拿到字节流 → SmartPLC 写 MinIO → 把 `file[].path`(MinIO URL)交给算法侧自取(文件不随请求上传)。这条链路要确认 MinIO 也配好(见 ⑥)。否则走旧上传链路。
5. **判定成功条件**:HTTP 200 **且** body `code==200` 才算成功;结果按 `positive/OK→OK`、`negative/NG→NG` 映射,按 (PointKey, StageIndex) 路由回对应通道。
6. **调试开关现场禁开**:`AlgorithmResultAlwaysOk`、以及 `mode=1`(全 OK)/`mode=2`(全 NG)只用于调试,试运行验证完务必关回 `mode=0`(实际)。

### MinIO(走直调链路时)

| 配置项 | 默认 | 现场 |
|---|---|---|
| `MinIOEndpoint` / `AccessKey` / `SecretKey` | (留空 / 现场配置) | 后端给 |
| `MinIOBucket` | `audio`(空时回落 `audio`) | 一般不改 |
| `MinIOUseSSL` | `False` | 按后端是否 TLS |

上传前会检查并按需建桶;写桶检查异常只记 Warn 并继续尝试,不中断主流程。

---

## ⑥ MySQL 连接(SmartQuality)

SmartQuality 直连 MySQL,无开关、始终启用 —— 它是配置(工位 / 检测点 / 盒子 / 灵敏度)和结果落库的来源,连不上整套跑不起来,现场必须先确认这条。

| 配置项 | 默认 |
|---|---|
| `MySqlHost` / `MySqlPort` | `localhost` / `3306` |
| `MySqlDatabase` | `bestplc` |
| `MySqlUsername` | `root` |
| `MySqlPassword` | (留空 / 现场配置) |

> 多阶段结果走事务一致性写入,每个方法调用会把 SQL 操作留痕到数据库调用记录,排查时可查。

---

## ⑦ 连接管理与重连(现场常见疑问)

| 机制 | 行为 |
|---|---|
| **标准连接** | 一个工位一个常驻连接,工作流主链路用 |
| **动态连接池** | 动态点位指定独立 PLC 地址时,按 `协议:连接串` 为键复用 / 新建 |
| **自动重连** | 内部定时器每 30 秒巡检,发现断连(标准 + 动态)就尝试重连 |
| **连接 / 接收超时** | Siemens / 欧姆龙 / Modbus 连接超时 5000ms、接收超时 10000ms |

现场拔网线 / 重启 PLC 后,不必手动重连 —— 等下一轮 30 秒巡检自动拉起。

---

## ⑧ 试运行 + 看日志 / 告警排错

配完一轮,过单实例与日志后做试运行:

检查清单:

1. **单实例**:程序靠 Mutex 保证同一时刻只有一个实例,重复启动会弹框退出,属正常。
2. **无产线自检**:产线还没排产时,可用 `AutoTest` 模式空跑验证链路(`AutoTestDuration` / `AutoTestStageCount` / `AutoTestInterval` 控时长 / 阶段数 / 间隔)。
3. **看分目录日志**(`App.config` 配的 log4net,按日滚动 `yyyyMMdd.log`):
   - `Log\StableWorkflow\` —— 工位主流程 / PLC 读写
   - `Log\StableWorkChildflow\` —— 单次检测 / 录音 / 算法
   - `Log\HallSignal\` —— 霍尔信号(含 Raw CSV),启用霍尔时看这里
   - `Log\General\` —— 通用
4. **常见现象 → 先查哪里**:

| 现象 | 优先怀疑 |
|---|---|
| PLC 读数离谱 / 浮点 NaN | `ModbusByteOrder` 字节序配错(③) |
| 整型检测值少了零头(如 484 而非 485) | 动态点 `valueTransform` 取整 / 缩放(②) |
| 连不上 PLC、反复重连 | 端口 / `protocol` / `rack`-`slot` / 节点号,等 30 秒巡检(⑦) |
| 录音空、无波形 | `EnableMicrophone`、盒子 IP/token、`ENV` 是否 `dev`(④) |
| 算法不返回 / 找不到模型 | `AlgorithmApiKey`、`AlgorithmFlowNumber` / 版本(⑤) |
| 通道结果错位 | detect_point.channel 与盒子物理声道接反(②④) |

5. GC / 告警有 `GcMonitorService` + `AlertLogService` 兜底记录,卡顿类问题交给性能诊断 skill(`smartplc-perf`)进一步分析。

---

## 延伸阅读

- 协议选型 / 点位模型 / 连接池细节 → [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)
- 录音盒子 / 算法 / MySQL / MinIO 集成细节 → [../reference/06-integrations.md](../reference/06-integrations.md)
- 全部配置项 + 默认值 + 外部连接清单 → [../reference/08-deployment-config.md](../reference/08-deployment-config.md)
- 库内先配工位 / 检测点 / 灵敏度 → [./01-line-operator-setup.md](./01-line-operator-setup.md)
- 名词不熟(动态点位 / 通道 / dataContent / 霍尔信号) → [../glossary/terms.md](../glossary/terms.md)

# 04 · PLC 多协议适配

> 事实来源:`PlcAdapter/`(`IPlc.cs`、`ProtocolFactory.cs`、`MultiProtocolAdapter.cs`、`ProtocolConnectionManager.cs`、`Siemens/Siemens1500.cs`、`Modbus/ModbusTcpAdapter.cs`+`ModbusTCP.cs`、`Omron/OmronCip.cs`+`OmronCipPlcAdapter.cs`)、`Docs/MultiProtocolSupport.md`、`Docs/ModbusHighLowByteConfig.md`、`Models/PlcConfigDynamicPoint.cs`、`Server/StableWorkChildflow.cs`。

## 为什么要多协议

SmartPLC 是中控,要插进客户**已经建好的产线**。而产线 PLC 的品牌由客户的设备商决定,现场什么都可能遇到:有的线全是西门子,有的是欧姆龙,还有一堆第三方设备只暴露 Modbus 寄存器。

如果中控只认一种协议,每进一个新客户就得改一次通信代码——这是现场实施成本的大头。SmartPLC 的做法是:把"用什么协议连 PLC"做成**一个配置项**,通信层按配置自动切换,点位表照填,上层工作流完全无感。这是 SmartPLC 的现场核心卖点之一。

> 底层三种协议**统一用 HslCommunication 工业通信库**(`SiemensS7Net` / `ModbusTcpNet` / `OmronFinsNet`),省去自己实现报文编解码。

## 三协议对比

| 维度 | Siemens S7-1500 | Modbus TCP | 欧姆龙(FINS TCP) |
|---|---|---|---|
| `protocol` 取值 | `siemens` / `s7` | `modbus` / `modbustcp` | `omron` / `omronfins` / `fins` / `finstcp` |
| 底层客户端 | `SiemensS7Net(S1500)` | `ModbusTcpNet` | `OmronFinsNet` |
| 默认端口 | 102 | 502 | 9600 |
| 寻址方式 | S7 地址(`DB`/`Rack`/`Slot`、字节偏移) | 寄存器地址(整数,`AddressStartWithZero=true`) | 内存区地址字符串(`D100`/`W100`/`H100`/`A100`/`C100`/`T100`) |
| 特有参数 | `rack`、`slot` | `station`、字节序(`byteOrder`) | `sa1`、`da1`(FINS 节点号) |
| 典型场景 | 西门子为主的产线、新建大线 | 第三方设备 / 仪表 / 杂牌 PLC,只给寄存器 | 欧姆龙系产线(CJ/CS/CP 系列) |

> 命名注意:欧姆龙实现类叫 `OmronCip`,但内部用的是 **FINS TCP** 协议(`OmronFinsNet`),不是 EtherNet/IP CIP。`Cip` 只是历史命名。术语见 [glossary/terms.md](../glossary/terms.md)。

### 字节序(Modbus / 欧姆龙)

多字节类型(short/int/float)的字节序由 `ModbusByteOrder` 配置决定,四档:`ABCD`(大端)、`BADC`、`CDAB`(字交换,默认)、`DCBA`(小端)。读数明显异常(整数离谱、浮点 NaN)优先怀疑字节序配错。细节与排查见 `Docs/ModbusHighLowByteConfig.md`。

## 统一适配层

三个协议被收敛到**同一个泛型接口** `IPlc<T>`(`Connect` / `Read` / `Write` / `WriteAsync`)。但上层主工作流 `StableWorkflow` 历史上只认 `IPlc<SiemensPlcPoint>`,所以有一层适配器把"任意协议"包装成"西门子点位接口":

```
StableWorkflow (只认 IPlc<SiemensPlcPoint>)
        │
        ▼
MultiProtocolAdapter : IPlc<SiemensPlcPoint>   ← 非西门子协议时包一层
        │  ConvertToGenericPoint:SiemensPlcPoint → ModbusTCPPoint / OmronCipPoint
        ▼
IPlc<IPlcPoint>  ← ModbusPlcAdapter / OmronCipPlcAdapter
        │
        ▼
HslCommunication 客户端(S7 / Modbus / FINS)
```

| 组件 | 职责 |
|---|---|
| `IPlc<T>` | 协议无关的读写接口(`PlcAdapter/IPlc.cs`) |
| `ProtocolFactory` | 工厂:按 `protocol` 字符串 `switch`,`CreateConnection` 造连接、`CreatePoint`/`CreateDynamicPoint` 造点位 |
| `MultiProtocolAdapter` | 把 `IPlc<IPlcPoint>` 适配成 `IPlc<SiemensPlcPoint>`,运行时把西门子点位转成目标协议点位 |
| `ModbusPlcAdapter` / `OmronCipPlcAdapter` | 把各自客户端包成统一 `IPlc<IPlcPoint>` |

> **向后兼容**:配置里没写 `protocol` 字段时,默认按 `Siemens` 处理(`ProtocolFactory` 与 `MultiProtocolAdapter` 都默认 Siemens)。老配置无需改动。
>
> `protocol` 是西门子时不套适配器,直接用 `Siemens1500`,少一层转换开销。

## 连接管理

`ProtocolConnectionManager` 统一管 PLC 连接,分两类:

| 连接类型 | 说明 |
|---|---|
| **标准连接** | 一个工位一个常驻连接(`InitializeStandardConnection`),工作流主链路用它 |
| **动态连接池** | 按 `协议:连接串` 为键的 `ConcurrentDictionary`,动态点位若指定了独立 PLC 地址,按键复用/新建 |

要点:

- **连接池**:动态连接按 `connectionString` 分组缓存;`GetDynamicConnection` 用信号量 + 双重检查锁定,防止并发重复建连。
- **自动重连**:内部一个 `Timer` 每 **30 秒**巡检一次,发现 `IsConnected == false` 的连接(标准 + 动态)就尝试重连。
- **连接 / 接收超时**(HslCommunication `PipeTcpNet`):
  - Siemens / 欧姆龙:连接超时 **5000ms**,接收超时 **10000ms**,持久连接(`IsPersistentConnection=true`)。
  - Modbus(`ModbusTcpNet`):连接超时 **5000ms**,接收超时 **10000ms**。
- **释放**:`Dispose` 时停定时器、关标准连接、清空动态连接池与连接锁。

## 点位模型

点位分两种,都由 `ProtocolFactory` 按协议造出对应类型(`SiemensPlcPoint` / `ModbusTCPPoint` / `OmronCipPoint`):

| 类型 | 含义 | 何时用 |
|---|---|---|
| **静态点位** | 配置里固定的读写点(心跳、信号、结果回写位等),`CreatePoint` 创建 | 工作流主链路的固定信号 |
| **动态点位** | 运行时按流程节点(`flowNode`)/阶段(`stage`)/通道(`channel`)写入的点,`CreateDynamicPoint` 创建 | 把检测值、阶段结果等下发到 PLC |

### 动态点位的值变换(valueTransform)

动态点位带一个可选的**值变换表达式** `valueTransform`(`Models/PlcConfigDynamicPoint.cs`),用于在写入前做单运算缩放/平移。支持四种,首字符是运算符,后面是操作数:

| 表达式 | 含义 | 例 |
|---|---|---|
| `*N` | 乘 | `*100`:把 `4.85` 写成 `485`(小数转定点整数) |
| `/N` | 除(除数为 0 时跳过变换) | `/10` |
| `+N` / `-N` | 加 / 减(偏移) | `+100` |

变换由 `StableWorkChildflow.ApplyValueTransform` 实现;表达式为空或非法时原值返回。

> **整数寄存器写入用四舍五入,不截断**。写 `Int`/`Short` 型动态点时,变换结果用 `Math.Round(..., MidpointRounding.AwayFromZero)` 取整,再转目标类型——避免浮点误差吃掉精度(代码注释举例:`4.85*100 = 484.999...`,直接强转截断会丢 `0.01`)。`Float` 型则直接 `(float)` 转换,不取整。

## 延伸阅读

- 这些点位在检测流程里怎么被读写 → [05-detection-workflow.md](./05-detection-workflow.md)
- 协议/点位之外的对外集成(盒子 / 算法 / DB / MinIO)→ [06-integrations.md](./06-integrations.md)
- 现场怎么选协议、配点位、联调 → [scenarios/03-field-implementation.md](../scenarios/03-field-implementation.md)
- 概念不熟(检测点 / 通道 / 动态点位)→ [03-key-concepts.md](./03-key-concepts.md) 或 [glossary/terms.md](../glossary/terms.md)

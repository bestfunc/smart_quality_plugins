# FAQ · 客户 / 评估者常问(对外)

> 事实来源:[../reference/01-product-overview.md](../reference/01-product-overview.md)、[04-plc-protocols.md](../reference/04-plc-protocols.md)、[08-deployment-config.md](../reference/08-deployment-config.md)、[09-target-customers.md](../reference/09-target-customers.md)、[11-roadmap.md](../reference/11-roadmap.md)。
>
> 本页是**对外口径**:回答客户与评估者最常问的问题,只讲系统现状能力,不承诺数字、不夸大、不涉内部细节。卖点话术(内部)见 [03-sales-talking-points.md](./03-sales-talking-points.md)。

---

## 一、它是什么 / 能不能上我这条线

### Q1. SmartPLC 到底是什么?

一套**装在产线工控机上的 Windows 桌面应用**(WPF / .NET 8),做**多工位声学质检自动化**。它站在产线 PLC 和 AI 算法之间当"中控":一头用工业协议实时读写 PLC 信号,一头驱动录音盒子采音、送深度学习算法判良品 / 不良品(OK / NG),再把结果写回 PLC 控制产品流转。详见 [../reference/01-product-overview.md](../reference/01-product-overview.md)。

### Q2. 我这条产线适不适合用?

看三个硬条件能不能**同时**成立:

1. 产线**已经上了 PLC 自动化**(系统靠读写 PLC 信号驱动);
2. 产品的**良 / 劣可以靠声音区分**(核心手段是录音 + 算法);
3. **多工位、按节拍流转**(系统按工位并发设计)。

三条都点头就值得往下评估。完整的适用 / 不适用判断见 [../reference/09-target-customers.md](../reference/09-target-customers.md)。

### Q3. 哪些情况明确不适合?

- 缺陷只在外观 / 尺寸 / 颜色上,**声音里没信息**(那是机器视觉的活);
- **没有 PLC、纯人工产线**(没有信号可读写);
- PLC 协议**不在支持的三种之内**(需另行评估适配工作量);
- 单件 / 单工位手工台(收益低);
- 环境噪声压过产品声、拾音信噪比太差(需先做声学隔离)。

---

## 二、PLC 兼容性

### Q4. 支持哪些 PLC?

现状适配三种主流工业协议(底层统一用 HslCommunication 通信库):

| 协议 | 取值关键字 | 默认端口 | 典型场景 |
|---|---|---|---|
| **Siemens S7-1500** | `siemens` / `s7` | 102 | 西门子为主的产线、新建大线 |
| **Modbus TCP** | `modbus` / `modbustcp` | 502 | 第三方设备 / 仪表 / 只给寄存器的杂牌 PLC |
| **欧姆龙 FINS TCP** | `omron` / `fins` 等 | 9600 | 欧姆龙系产线(CJ/CS/CP 系列) |

"用哪种协议连 PLC"是**一个配置项**,通信层按配置自动切换,点位表照填,上层流程无感。明细与点位模型见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)。

### Q5. 我的 PLC 不在这三种里怎么办?

现状只覆盖上面三种协议。其他品牌 / 协议需要**单独评估适配工作量**,不在开箱即用范围内——这一点会如实告知,不打包票。更多 PLC 品牌属于演进设想里被列为"有需求再加"的事项,当前不在计划内(见 [../reference/11-roadmap.md](../reference/11-roadmap.md))。

### Q6. 接入要不要改造现有产线?

不需要重做自动化。前提是客户能提供**点位表**、PLC 协议在支持的三种之内、且点位可读写。系统按点位表配置即可对接,改动集中在中控侧的配置,不要求改 PLC 程序逻辑。现场对接流程见 [../scenarios/03-field-implementation.md](../scenarios/03-field-implementation.md)。

---

## 三、部署与上云

### Q7. 怎么部署?

**产线现场单机部署**:一个 WPF / .NET 8 桌面应用,装在产线工控机(Windows)上,承载全部逻辑。一机一线——一台中控对应一条产线,多条产线各自一台工控机独立部署。无容器化、无独立后端服务进程。部署形态与运行环境见 [../reference/08-deployment-config.md](../reference/08-deployment-config.md)。

### Q8. 能不能上云 / SaaS 化?

现状是产线现场单机,**不是云端 / SaaS 形态**。集中式、服务化的方向在演进路线图里(代号 v4 / SmartPLCV3 的后台服务化重构设想),但那是**方向性设想、非交付承诺**,具体范围与节奏以实际推进为准。详见 [../reference/11-roadmap.md](../reference/11-roadmap.md)。

### Q9. 中控机要连哪些东西?

现场上线前确认这几条连接(地址 / 密钥均现场配置):

| 服务 | 用途 |
|---|---|
| 产线 PLC(S7 / Modbus / 欧姆龙 FINS TCP) | 读信号、回写结果 |
| 录音盒子(SmartAudio) | 采集音频,一盒多通道 |
| 算法引擎(HTTP/JSON) | 接收音频返回 OK/NG |
| MySQL | 配置 + 业务数据落库 |
| MinIO | 存音频文件 |

清单见 [../reference/08-deployment-config.md](../reference/08-deployment-config.md) 的"外部服务连接清单"。

---

## 四、数据与安全

### Q10. 数据存在哪?安不安全?

- **业务数据 / 配置** → MySQL(默认库名 `bestplc`);
- **音频文件** → MinIO(默认桶 `audio`)。

两者地址、账号、密钥都**由现场配置**,可放在客户内网,不强制外联公网。系统内置登录认证、关停程序密码保护、操作审计等安全机制。每个检测动作(音频调用、算法结果、数据库操作、告警)全程留痕,可追溯。具体配置项见 [../reference/08-deployment-config.md](../reference/08-deployment-config.md)。

### Q11. 数据会不会传到厂商那边?

部署形态是现场单机,MySQL / MinIO / 算法服务的地址均现场配置,可全部落在客户自有环境内。本页不替任何现场承诺网络边界——具体数据流向以实际部署的配置为准。

---

## 五、检测效果

### Q12. 检测准确率有多少?

**这取决于算法对该产品声学特征的建模质量,不给一个普适的数字。** 算法对某产品"听不听得懂",由采样数据量与训练效果决定;同一套系统,不同产品、不同声学环境下的表现会不一样。任何脱离具体产品和现场的"准确率承诺"都不可信——这一点对外要讲清楚,不编数字。建模相关前提见 [../reference/09-target-customers.md](../reference/09-target-customers.md) 第④节。

### Q13. 判得太松 / 太严能调吗?

能。每个检测点有**灵敏度**参数,可按产品差异化设置,质量工程师据 OK/NG 分布收紧或放宽判定。调灵敏度的流程见 [../scenarios/02-quality-engineer-review.md](../scenarios/02-quality-engineer-review.md)。

---

## 六、稳定性与运维

### Q14. 盒子 / 中控重启会怎样?

系统是单进程多线程的常驻应用,设有单实例互斥(同一时刻只允许一个程序实例)。重启后按配置重新建立 PLC、录音盒子、数据库等连接。PLC 连接内部有自动重连机制(定时巡检掉线连接并重连),细节见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md) 的连接管理段。

### Q15. 断网 / PLC 掉线了怎么办?

PLC 连接管理器内部有定时巡检,发现连接断开会尝试重连;连接 / 接收都有超时设置,不会无限期挂死。对算法、数据库等外部服务的调用也都带超时。具体超时值与重连周期见 [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)。检测过程中若某环节卡住(算法 / 盒子 / PLC 无响应),主流程有**操作超时兜底**:超过 `OperationTimeOut`(默认 30 秒)会把在产工件判为 **Timeout** 并自动复位工作流、通过系统状态点位通知 PLC,不会无限挂起;PLC 掉线则停止轮询,交由连接管理器重连后恢复。

### Q16. 支持多少个工位?

系统按"**每工位一个常驻主流程**"的模型并发驱动,单台中控对应一条产线、承载该线的多个工位。系统设计上支持多工位独立并发(线程池预热、服务端 GC 等做了支撑)。具体一台中控能稳定带多少工位,取决于现场硬件、产品节拍与单次检测耗时,需结合实际产线评估,不给一个固定上限。

---

## 七、接入周期

### Q17. 一个新产品接入要多久?

不是开箱即用。新品类首次接入通常要经历一个**导入期**:采样录音 → 算法训练建模 → 现场联调 → 调灵敏度。这段工程量在落地之前,周期取决于产品声学特征复杂度、采样数据量与现场配合度。本页不给一个统一天数——如实说明"有建模导入期"比报一个空数字更负责。前提条件见 [../reference/09-target-customers.md](../reference/09-target-customers.md) 第④节。

### Q18. 现场上线大致要做哪些事?

可粗分四步:① 配工位 / PLC 点位 / 设备地址;② 联调录音盒子 + 算法引擎;③ 采样建模、调灵敏度;④ 试跑验证后转量产。完整步骤见 [../scenarios/01-line-operator-setup.md](../scenarios/01-line-operator-setup.md) 与 [../scenarios/03-field-implementation.md](../scenarios/03-field-implementation.md)。

---

## 延伸阅读

- 它是什么、解决什么问题 → [../reference/01-product-overview.md](../reference/01-product-overview.md)
- 支持哪些 PLC(兼容矩阵)→ [../reference/04-plc-protocols.md](../reference/04-plc-protocols.md)
- 怎么部署、配置在哪填 → [../reference/08-deployment-config.md](../reference/08-deployment-config.md)
- 适不适合我这条线 → [../reference/09-target-customers.md](../reference/09-target-customers.md)
- 未来会往哪走(设想)→ [../reference/11-roadmap.md](../reference/11-roadmap.md)
- 名词不熟 → [../glossary/terms.md](../glossary/terms.md)

# 场景 02 · 质量工程师:查检测结果、分析 OK/NG、调灵敏度

> 事实来源:`../reference/03-key-concepts.md`、`../reference/05-detection-workflow.md`、`../reference/06-integrations.md`(本篇为场景化串联,事实以这三篇为准)。

## ① 场景设定

你是产线的**质量工程师**。SmartPLC 已经在工控机上跑起来,产品在工位间自动流转、自动判 OK/NG。你的日常不是配置系统,而是:

- 盯当班的 OK/NG 分布,看有没有异常批次;
- 抽查可疑工件——尤其客户投诉、或某工位 NG 率突然抬头时;
- 判断一次 NG 是**真不良**还是**误判**,再决定要不要动检测点的灵敏度;
- 把历史检测数据导出给品质例会 / 客户报告。

你不碰 PLC 点位表,也不改协议(那是实施 / 运维的活,见 [01-line-operator-setup.md](./01-line-operator-setup.md) 与 [03-field-implementation.md](./03-field-implementation.md))。你的杠杆只有一个:**每个检测点的灵敏度**,外加对结果数据的解读。

**先建立心智模型**(术语见 [../glossary/terms.md](../glossary/terms.md)):

```
检测工位 detect_station
  └─ 检测点 detect_point ── 灵敏度 sensitivity(你能调的那个旋钮)
        └─ 通道 channel(录音盒子声道 1-4)
一次检测 detect(产品级) → 阶段 detect_stage(1-8) → 通道 detect_channel(声道级结果)
```

一个产品来一次,系统按 1-8 个阶段独立录音 + 独立算法判定,最后聚合成产品级 OK/NG 写回 PLC。详见 [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md)。

---

## ② 在哪看结果

三个层面,从"看个大概"到"挖到底",由浅入深:

| 看什么 | 在哪看 | 适合 |
|---|---|---|
| 工位实时状态 / 进度 / 告警 | SmartPLC 工位监控页(UI) | 当班盯线、抓异常工位 |
| 单次检测有没有调音频、调算法、调库 | 音频调用日志 / 数据库调用记录(留痕) | 复盘一次具体检测怎么走的 |
| 结构化的 OK/NG 明细(产品 / 阶段 / 通道) | MySQL 业务库(`detect` / `detect_stage` / `detect_channel`) | 统计分析、误判定位、导出 |

- **工位监控页**:实时看所有工位的流程进度、PLC 读写、检测结果与告警,带心跳记录。先在这里发现"哪个工位不对"。
- **音频调用日志 / 数据库调用记录**:每次集成调用(录音、算法、落库)都留痕——`SmartQualityClientV2` 经 `RecordMethodCallAsync` 把 SQL 操作写进数据库调用记录(可观测兜底)。用来回答"这次检测到底调没调算法、调的什么"。
- **MySQL 业务库**:最终事实在这里。`SmartQualityClientV2` 直连 MySQL(默认库名 `bestplc`),核心表见下表。这是统计与导出的数据源。

| 表 | 你在这里看什么 |
|---|---|
| `detect` | 一次检测的**产品级**主记录:总 OK/NG、条码、产品、起止时间 |
| `detect_stage` | 该次检测的**阶段级**结果(1-8 个阶段) |
| `detect_channel` | **通道级**结果:`aiResult`(算法判定)、`fileID`(对应音频文件) |
| `detect_point` | 检测点配置——**`sensitivity` 就在这张表**,你要调的就是它 |
| `product` | 产品型号(灵敏度、检测点配置按 `productId` 区分) |

> 表与字段定义见 [../reference/06-integrations.md](../reference/06-integrations.md) §③。

---

## ③ 读懂 OK/NG 与阶段结果

**写回 PLC 的值约定**(看监控页 / 点位时对照):

| 值 | 含义 |
|---|---|
| `-1` | 未就绪 / 已复位(还没出结果) |
| `1` | OK(良品) |
| `2` | NG(不良) |

**结果是怎么聚合上来的**(自下而上,见 [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md) §结果聚合):

```
通道 detect_channel 各自的算法判定(aiResult)
   → 汇成阶段 detect_stage 结果(任一通道 NG → 该阶段 NG)
      → 汇成产品级 detect 综合结果 → 写 WPointResult(1/2)+ WPointErrorCode
```

阶段结果写到 `WPointStage1Result`..`WPointStage8Result`(阶段号到点位写死映射,1=OK / 2=NG)。

**读结果时容易踩的坑**:

- **看到全 OK 别急着高兴**:`AlgorithmResultAlwaysOk=true` 时,落库**保留真实结果**,但写回 PLC 的 `WPointResult` 被强制写 1(产线放行调试用)。所以**以库里的 `detect` / `detect_channel` 为准**,别只看产线放行信号。
- **看到 `Test` 结果**:`EnableAlgorithm=false`(没开算法)时,阶段 / 主结果落库为 `Test`、PLC 写默认 `Code=100`——这不是真判定,是空跑。
- **看到 `Code=50/timeout`**:算法结果等满 5 秒(`WaitAlgorithmResultTime` 默认)没回,系统写的兜底超时码。**这是性能 / 连通性问题,不是产品不良**,别当 NG 统计。
- **看到 `Timeout` / `Reset` 主记录**:整次操作超时(默认 30s)或现场复位,系统落了兜底记录,不是正常完成的检测。

---

## ④ 复核:回放音频

判一个 NG 真假,最直接的办法是**听那段音频**。

1. 在 `detect_channel` 找到这次 NG 的 `fileID`,定位对应音频文件。
2. 音频持久化在 **MinIO**(bucket 默认 `audio`,`contentType=audio/wav`);走算法直调链路时,音频由"盒子 → SmartPLC → MinIO"落地,算法侧再按 `file[].path`(MinIO URL)自取。
3. 实时盯线时,UI 的**实时听音**走 WebSocket(`SmartAudioClientStreamer`,`ws://{IP}:{Port}/api/v1/stream/live`,固定 44.1kHz / 16bit,通道 1-4)。

> 重要:**实时听音只服务 UI 波形 / 播放,不参与 OK/NG 判定**(见 [../reference/06-integrations.md](../reference/06-integrations.md) §①)。复核的依据是落库的算法结果 + MinIO 里那段已检测音频,不是实时流的主观听感。

**复核检查清单**:

- [ ] 这次 NG 对应哪几个阶段 / 哪个通道?(查 `detect_stage` / `detect_channel`)
- [ ] 取出 `fileID` 对应音频,人工听:确有异响 → 真 NG;声音正常 → 疑似误杀。
- [ ] 排除非产品因素:是不是 `Code=50/timeout`(算法没回)、`Test`(没开算法)、或环境噪声窜进通道?
- [ ] 同一产品 / 同一工位是不是成片 NG?成片误杀往往是灵敏度太紧或盒子 / 环境问题,不是单件产品的事。

---

## ⑤ 误判分析:太松漏检 vs 太紧误杀

灵敏度是一根**双向的弦**,松紧各有代价:

| 症状 | 方向 | 现场表现 | 风险 |
|---|---|---|---|
| **灵敏度太松** | 判定门槛低,轻微异响被放过 | NG 率偏低、产线很顺,但客户端 / 下游发现漏检的不良 | **漏检**——不良品流出,最贵的错误 |
| **灵敏度太紧** | 判定门槛高,正常波动也算异响 | NG 率突然抬高、复核一听大多正常 | **误杀**——好件被判 NG,良率虚低、产能损失 |

**判断到底是哪种,而不是凭感觉**:

1. 拉一段时间的 `detect` / `detect_channel`,算该产品 / 该工位的 NG 率与趋势。
2. 对 NG 件**抽样回放音频**(见 ④):误杀多 → 偏紧;漏检反馈多(NG 率低但有不良流出) → 偏松。
3. **定位到点**:NG 是不是集中在某个 `detect_point`(某通道 / 某阶段)?灵敏度是检测点级的,要落到具体点才好调。
4. 区分清楚"产品问题"和"系统问题":先排除 `timeout` / `Test` / 环境噪声,再谈灵敏度。

> 调灵敏度之前,**先确认是误判而不是系统故障**。`Code=50/timeout` 一片、某盒子掉线,这些调灵敏度也没用。

---

## ⑥ 调灵敏度

**事实**(来自 [../reference/03-key-concepts.md](../reference/03-key-concepts.md)):

- 灵敏度存在 `detect_point.sensitivity`,DB 类型 `smallint` **整数**,范围 **1-255**,默认 **220**。
- 它是**检测点级**的——同一工位不同检测点可以不同灵敏度。
- 它**按产品下发给盒子**:检测点配置与灵敏度按 `productId` 区分,所以可以**按产品差异化**。

**值的方向(经验性,需结合现场复核校准)**:数值大体表示判定的严格程度。调整方向:

- 漏检(太松)→ **调紧**(向收紧异响门槛的方向调);
- 误杀(太紧)→ **调松**;
- 一次只动一个产品 / 一个检测点的一档,别全线一把梭。

> 注意:SmartPLC 侧只是把 `sensitivity`(1-255)**透传**给算法 / 盒子,"数值调大对应更紧还是更松"的方向语义由**算法端定义**,不在本系统内。稳妥做法:**改一档 → 跑一批 → 回放复核**,以现场算法版本实际行为为准,比查表可靠。

**调灵敏度检查清单**:

- [ ] 已确认是**误判**(回放复核过),不是 timeout / Test / 盒子掉线;
- [ ] 已定位到**具体 `detect_point`**(哪个工位、哪个通道、哪个产品);
- [ ] 在 `detect_point.sensitivity` 改一档(1-255 整数),只动目标产品(`productId`);
- [ ] 灵敏度下发到盒子依赖 `EnableSensitivitySync`(默认 `False`)——**按产品同步是演进中的能力**,现场是否生效要和实施确认(见 [../reference/06-integrations.md](../reference/06-integrations.md) §⑤);
- [ ] 改完**跑一批**,再拉 `detect` / `detect_channel` 看 NG 率与误判有没有收敛;
- [ ] 记录"改了什么、为什么、效果"——方便回退和品质追溯。

> 灵敏度配置属于产线配置范畴,实际改值操作 / 配置入口见 [01-line-operator-setup.md](./01-line-operator-setup.md)。质量工程师的核心是**判断该往哪个方向调、调多少**。

---

## ⑦ 导出历史数据

品质例会、客户报告、批次追溯,数据源是 MySQL 业务库(`bestplc`)。按粒度选表:

| 你要的报告 | 主表 | 关联 |
|---|---|---|
| 批次 / 当班 OK/NG 汇总 | `detect`(产品级) | 按时间 / `productId` / 工位过滤 |
| 阶段不良分布(哪个阶段最容易 NG) | `detect_stage` | 关联 `detect` 取产品 / 批次 |
| 通道 / 声道级明细 + 音频溯源 | `detect_channel`(`aiResult` / `fileID`) | `fileID` 回查 MinIO 音频 |
| 单件追溯 | `detect` 的条码(barcode) | 一个工件实例的唯一标识 |

**导出注意**:

- 统计前**过滤掉非真实判定**:`Test`(没开算法)、`Timeout` / `Reset`(兜底记录)、`Code=50/timeout`(算法超时)——混进去会污染良率数据。
- `detect_channel.fileID` 是音频溯源的钥匙;要附音频证据的报告,带上 `fileID` → MinIO URL。
- 数据库连接 / 凭据由运维管理,质量工程师按只读账号取数即可(凭据不在本文档范围,见红线说明)。

---

## ⑧ 延伸阅读

- 工位 / 检测点 / 通道 / 阶段 / 灵敏度等概念 → [../reference/03-key-concepts.md](../reference/03-key-concepts.md)
- 8 阶段时序、结果如何聚合写回 → [../reference/05-detection-workflow.md](../reference/05-detection-workflow.md)
- 录音盒子 / 算法引擎 / MySQL / MinIO 对接细节与开关 → [../reference/06-integrations.md](../reference/06-integrations.md)
- 改灵敏度 / 配产线的实操 → [01-line-operator-setup.md](./01-line-operator-setup.md)
- 名词不熟(dataContent / aiResult / fileID / 灵敏度) → [../glossary/terms.md](../glossary/terms.md)

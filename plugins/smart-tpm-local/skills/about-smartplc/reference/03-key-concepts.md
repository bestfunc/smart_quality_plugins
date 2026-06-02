# 03 · 核心概念

> 读代码 / 配置 / 文档前先扫这一页。完整缩写与索引见 [glossary/terms.md](../glossary/terms.md)。

## 业务实体

| 概念 | 含义 | 出处 |
|---|---|---|
| **检测工位**(detect_station) | 产线上一个检测位置,对应一套 PLC + 设备配置 | `SmartQualityClientV2`、`PlcConfig` |
| **检测点**(detect_point) | 工位内一个具体检测位,绑定某个智能设备的某个通道,带灵敏度 | `Models/DetectPoint.cs` |
| **智能设备**(smart_device) | 录音盒子,一个 IP + 多个声道(通道),挂在工位下 | `PlcConfigSmartDevice` |
| **通道**(channel) | 录音盒子内部声道号(1-4),区分多路音频;直接对应 detect_point.channel | `DetectPoint.Channel` |
| **产品**(product) | 被检测的产品型号;检测点配置、灵敏度按产品(productId)区分 | `product` 表、`DetectPoint.productId` |
| **条码**(barcode) | 单个工件实例的唯一标识,用于数据追溯 | `ChildWorkflowInfo.Barcode` |

## 流程概念

| 概念 | 含义 | 出处 |
|---|---|---|
| **主流程**(StableWorkflow) | 一个工位的常驻引擎:周期读 PLC 心跳/信号,驱动子流程 | `Server/StableWorkflow.cs` |
| **子流程**(StableWorkChildflow) | 单个产品的一次完整检测,8 阶段状态机,检测完即销毁 | `Server/StableWorkChildflow.cs` |
| **阶段**(stage) | 一次检测内的步骤划分(1-8),每阶段独立录音 + 算法判定 | `WPointStage{1..8}Result` |
| **动态点位**(dynamic point) | 运行时按配置写入 PLC 的点位,支持值变换表达式(如 `*100`) | `Models/PlcConfigDynamicPoint.cs` |

## 检测参数

| 概念 | 含义 | 出处 |
|---|---|---|
| **灵敏度**(sensitivity) | 检测点的音频检测灵敏度,DB 为 `smallint` 整数,范围 1-255、默认 220;按产品下发给盒子 | `detect_point.sensitivity` |
| **算法引擎** | 音频检测算法后端,接收音频返回 OK/NG;端点 `/api/v2/detect/execute` | `Server/Algorithm/` |
| **dataContent** | 算法请求体里的嵌套数据结构(detect / detect_channel / detect_point / file 等) | `AlgorithmRequest.cs` |
| **霍尔信号** | 霍尔传感器脉冲,经 Modbus 读取,用于精确同步产品位置 | `ModbusHallReader`、`Docs/` 霍尔分析文档 |

## 概念关系

```
检测工位 (detect_station)
  ├─ 智能设备 (smart_device, 录音盒子) ×N
  │     └─ 通道 (channel 1-4)
  │           └─ 检测点 (detect_point) ── 灵敏度 (sensitivity)
  └─ 产品 (product) ── 决定该工位用哪套检测点配置

主流程 (StableWorkflow, 每工位一个)
  └─ 子流程 (StableWorkChildflow, 每个产品一次)
        └─ 阶段 (stage 1-8)
              └─ 录音(通道) → 算法 → OK/NG → 写回 PLC
```

> 灵敏度按产品下发给盒子这条链路有专门设计(产品切换 / 首次加载时差异化同步),属于演进中的能力,细节见相关设计文档。

## 延伸阅读

- 协议与点位 → [04-plc-protocols.md](./04-plc-protocols.md)
- 检测全流程时序 → [05-detection-workflow.md](./05-detection-workflow.md)
- 完整术语表 → [glossary/terms.md](../glossary/terms.md)

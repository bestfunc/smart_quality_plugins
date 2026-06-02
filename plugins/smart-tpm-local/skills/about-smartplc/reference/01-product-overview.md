# 01 · 产品概述

> 事实来源:`BestPLC/BestPLC.csproj`(Product=SmartPLC,Company=巅峰表现)、各 UI 页面、`Docs/` 文档。

## 它是什么

**SmartPLC**(代码库名 `BestPLC`,出品方"巅峰表现")是一套**装在产线工控机上的 Windows 桌面应用**(WPF / .NET 8),用于**工业产线多工位声学质检自动化**。

它站在产线 PLC 与 AI 算法之间做"中控":一头用工业协议实时读写 PLC 信号,一头驱动录音盒子采集音频、送深度学习算法判定良品 / 不良品(OK / NG),再把判定结果写回 PLC 控制产品流转。

## 解决什么问题

| 痛点 | SmartPLC 的应对 |
|---|---|
| **人工听检低效、易误检** | 用 AI 声学算法替代人耳,判定标准统一、可量化 |
| **多工位状态不可观测** | 实时监控每个工位的流程进度、PLC 读写、检测结果(见 [05-detection-workflow.md](./05-detection-workflow.md)) |
| **检测动作难追溯** | 音频调用、算法结果、数据库操作、告警全程留痕(见 [06-integrations.md](./06-integrations.md)) |
| **多协议产线对接成本高** | 统一适配 Siemens S7-1500 / Modbus TCP / 欧姆龙 FINS TCP(见 [04-plc-protocols.md](./04-plc-protocols.md)) |

## 核心价值

1. **降低次品流出**:声学算法判定比人耳一致性高、可复核。
2. **提高产能**:多工位自动流转,检测环节无人值守。
3. **可观测、可追溯**:每个检测动作有结构化日志,问题可根因分析。
4. **现场适配快**:三种主流 PLC 协议开箱适配,按点位表配置即可上线。

## 谁在用

| 角色 | 主要诉求 | 相关页面 |
|---|---|---|
| **产线运维** | 配置工位、PLC 点位、设备地址、检测灵敏度 | 见 [scenarios/01-line-operator-setup.md](../scenarios/01-line-operator-setup.md) |
| **质量工程师** | 查检测结果、分析 OK/NG 分布、调灵敏度 | 见 [scenarios/02-quality-engineer-review.md](../scenarios/02-quality-engineer-review.md) |
| **现场实施** | 对接客户 PLC、按点位表配点、联调盒子 + 算法 | 见 [scenarios/03-field-implementation.md](../scenarios/03-field-implementation.md) |
| **系统管理员** | 用户认证、日志审计、系统监控 | — |

## 核心功能(业务视角)

| 功能域 | 能做什么 |
|---|---|
| **PLC 通信与工位控制** | 多协议适配、点位实时读写、多工位独立并发、霍尔脉冲同步 |
| **AI 声学检测** | 录音盒子采音、音频送算法、结果关联产品/工位/阶段、灵敏度按产品下发 |
| **工作流引擎** | 待检 → 检测中 → 判定 → 流转 的状态机,8 阶段动作链(读 PLC → 录音 → 算法 → 写回) |
| **多工位监控** | 实时看所有工位状态/进度/告警,心跳记录,性能诊断(GC、延迟) |
| **数据与日志** | 音频调用日志、告警日志、数据库调用记录、历史数据导出 |
| **配置管理** | 设备/工位/点位映射、动态点位、智能设备(多声道)配置 |
| **安全审计** | 登录认证、关闭程序密码保护、操作审计 |

## 部署形态

**产线现场单机**:单一 WPF 应用运行在产线工控机(Windows),承载全部逻辑,无容器化。一台中控连接本线的 PLC、录音盒子、以及后端的 MySQL / MinIO / 算法服务。

```
产线
 ├─ PLC #1 + 录音盒子 + 霍尔传感器
 ├─ PLC #2 + 录音盒子 + 霍尔传感器
 └─ ...
        ↓ (S7 / Modbus / 欧姆龙)
   SmartPLC 中控(本机)
        ├─→ 算法引擎(HTTP,深度学习 OK/NG)
        ├─→ MySQL(配置 + 业务数据)
        └─→ MinIO(音频文件)
```

配置项详见 [08-deployment-config.md](./08-deployment-config.md)。系统的演进方向(工位微服务化)见 [11-roadmap.md](./11-roadmap.md)。

## 延伸阅读

- 想看系统怎么搭起来的 → [02-architecture.md](./02-architecture.md)
- 想懂术语 → [03-key-concepts.md](./03-key-concepts.md)
- 想看一个产品怎么被检测的 → [05-detection-workflow.md](./05-detection-workflow.md)

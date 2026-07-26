# 场景：接手人上手工作流

面向"刚接手 SmartQuality 某个系统的开发/新员工"。目标是把"该从哪读起、怎么跑通"收敛成一条可执行路径。总览见 [reference/02-architecture.md](../reference/02-architecture.md)，黑话见 [glossary](../glossary/terms.md)。

## 第一步：建立全局观（半天）

先别急着跑代码，先搞清"我负责的系统在整条链路的哪一段"：

1. 读 [reference/01-product-overview.md](../reference/01-product-overview.md) —— 产品由哪些系统组成。
2. 读 [reference/02-architecture.md](../reference/02-architecture.md) —— 端到端数据流。
3. 读 [reference/04-key-concepts.md](../reference/04-key-concepts.md) —— 至少先认全这几个词：检测流程 detect_flow、算法流 algorithm_flow、数据集 dataset、粗标/细标、工位分库、块 block、mode(default/sqMode)。

## 第二步：定位你的系统（按分工二选一）

| 你被分到 | 直接读 | 先跑起来的最短路径 |
|---|---|---|
| Web 检测平台 | [systems/05-web-platform.md](../systems/05-web-platform.md) | docker-compose 起后端依赖（MySQL/Redis/InfluxDB/MinIO），前端 `npm run dev` |
| 数据同步服务端 | [systems/02-data-sync-service.md](../systems/02-data-sync-service.md) | 本地起目标库 + 服务端，跑一次 gRPC 收块 |
| 数据同步客户端 | [systems/03-data-sync-client.md](../systems/03-data-sync-client.md) | 用 `config.sqmode.yaml` 跑 sqMode（无源端依赖）最快 |
| 安卓采集端 | [systems/04-app-client.md](../systems/04-app-client.md) | Flutter 工程在 `smart_tpm_app` 的 `feature/app-v2` 分支；出 APK |
| 桌面采集端 | [systems/04-app-client.md](../systems/04-app-client.md) | `SmartQuality-client`，Electron，`electron-builder.yml` 出包 |
| 微信服务平台 | [systems/01-wechat-service.md](../systems/01-wechat-service.md) | Flask 后端 docker-compose 起，先跑通会话存档拉取 |
| 移动套装 / AI 插件 | [systems/06-mobile-suite.md](../systems/06-mobile-suite.md) | 装 smart_quality_plugins 插件，OAuth 授权跑第一个 MCP skill |

每篇 `systems/` 文档末尾都有"接手人从哪读起代码"清单——按那个顺序读入口文件，别一上来通读全仓。

## 第三步：跑通"一条真实链路"（关键里程碑）

理解一个系统最快的方式是让数据流过它一次。推荐先跑通最短的一条：

- **纯采集→同步**：本地用 sqMode 客户端 + 本地服务端，手动放一条 jsonl + 一个音频到共享目录，看它进目标库。（见 [systems/03](../systems/03-data-sync-client.md)）
- **平台内闭环**：在 Web 平台建一个检测流程，跑一个测试任务，看它产出测试记录。（见 [systems/05](../systems/05-web-platform.md)）
- **完整链路**：见 [scenarios/02-end-to-end-dataflow.md](./02-end-to-end-dataflow.md)。

## 第四步：接手交接物（若你是接替离职同事）

对照 [faq/02-handover-crosswalk.md](../faq/02-handover-crosswalk.md) 逐项确认：
- 代码分支对不对（尤其安卓在 `feature/app-v2`、Web 在 `dev`、同步在各自主线）；
- 凭据是否通过凭据渠道拿到（不在代码/文档里）；
- 每个系统的"已知缺陷"你是否都过了一遍（那是踩坑地图）。

## 常见起步坑

- **看错分支**：几个仓库的活跃主线不是 master（安卓在 `feature/app-v2`、Web/bi 在 `dev`）。先 `git branch -r` 看哪条分支提交最多最新。
- **配置含密钥**：各仓库 `config.*.yaml` / `.env` 里的真实密钥走凭据渠道，不在公开仓库；本地跑用自己的测试值。
- **文档漂移**：`sync-data` README 与实现有出入（见 [systems/03](../systems/03-data-sync-client.md) 首部提醒），以代码为准。

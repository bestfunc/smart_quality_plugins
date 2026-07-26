# FAQ：接手开发常见问题

面向接手 SmartQuality 系统的开发。术语见 [glossary](../glossary/terms.md)。

## 命名与范围

**Q：SmartQuality、Smart TPM、"检测平台"是几个东西？**
一个。同一产品套装的不同叫法。仓库前缀有 `SmartQuality-*` 也有 `smart_tpm_*`，历史遗留，指同一产品。

**Q：这产品到底解决什么问题？**
产线上原本靠人耳听产品声音判合格与否，本产品用麦克风/声学盒子录音 + 算法自动判 **OK/NG**，并把数据沉淀成数据集用于复核和算法迭代。

**Q：这份百科和 about-smart-tpm-mcp / about-audio-quality / about-net-audio 什么关系？**
本百科讲**产品全貌 + 交接**；那几个是**子百科**，各讲一个细分层（MCP 集成层 / 声学后端 / 联网录音）。查全局看这里，查某层细节去子百科。见 [reference/12-ecosystem.md](../reference/12-ecosystem.md)。

## 代码在哪 / 哪个分支

**Q：每个系统对应哪个仓库？**
见 [faq/02-handover-crosswalk.md](./02-handover-crosswalk.md) 的对照表。

**Q：为什么有的仓库 master 很旧？**
几个仓库的活跃主线不是 master：
- `smart_tpm_app`（安卓）活跃在 **`feature/app-v2`**，master 停在一年前；
- `smart_tpm_web_v2`、`bi` 活跃在 **`dev`**；
- `sync-data` 主线在 **`main`**。
接手时先 `git branch -r` + 看各远程分支提交数，最多最新的那条才是真主线。

**Q：安卓端是 Flutter 还是 Electron？**
安卓端 `smart_tpm_app` 是 **Flutter/Dart**，出 APK。`SmartQuality-client`（smart_quality_app_v2）是 **Electron + React** 桌面端（`electron-builder.yml` 佐证），只出 Win/macOS/Linux，**不打安卓包**。两套独立代码，仅共享声学盒子硬件协议。详见 [systems/04](../systems/04-app-client.md)。

## 配置与凭据

**Q：config.yaml / .env 里的密钥能提交吗？**
不能。真实密钥、口令、内网 IP 走**凭据渠道**单独移交（协议第 4.3.4 款），不进公开仓库。本地开发用自己的测试值，用 `${ENV:default}` 注入。

**Q：一个同步客户端换个现场部署要改什么？**
主要改该现场的 `config.*.yaml`：`mode`、`client_id`（全局唯一）、源端连接或共享目录、`workstation_id`。目标库名/bucket/同步开关不在客户端配，由服务端 Admin 按 `client_id` 下发。见 [systems/03](../systems/03-data-sync-client.md)。

## 已知文档待校对（诚实标注）

> 这些是撰写时发现的、代码与文档/彼此之间不完全一致的地方，接手后请以**代码为唯一事实来源**核实修订：

| 项 | 情况 | 以什么为准 |
|---|---|---|
| sync-data README 漂移 | README 称本地存储用 SQLite(WAL)、有 Binlog/Redis Stream/16分片等；实际 sync-client 是文件块+轮询扫描，服务端也未实现那套 | 实际 `internal/` 目录与源码（见 [systems/02](../systems/02-data-sync-service.md)、[systems/03](../systems/03-data-sync-client.md) 首部提醒） |
| SmartQuality-client 技术栈表述 | 各系统文档对桌面端框架描述需二次确认 | 仓库根 `electron-builder.yml` / `electron/` 目录 → 以 **Electron** 为准 |
| 微信→云效自动建工作项 | README 宣称的主线因 `yunxiao_bp` 在 `app.py` 被注释而**当前未运行** | 代码（见 [systems/01](../systems/01-wechat-service.md)） |
| MCP tool 数量 | 文档写"9 模块 61 tool" | 以后端 `tools/list` 实际返回为准 |
| 安卓 release 签名 | 当前仍用 debug 签名、包名为脚手架默认 | 待整改项，正式签名需配 keystore（走凭据渠道） |

## 排障入口

**Q：跨系统数据"卡住了"从哪查起？**
按 [scenarios/02-end-to-end-dataflow.md](../scenarios/02-end-to-end-dataflow.md) 的五段链路定位：采集 → 上云 → 同步 → 检测/标注 → 分析。先确认数据卡在哪一段，再翻对应 `systems/` 文档的"日志与故障排查"。

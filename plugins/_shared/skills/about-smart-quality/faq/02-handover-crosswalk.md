# 交接对照表（Handover Crosswalk）

把"协商解除劳动合同协议"第四条列出的交接项，映射到**仓库 → 本百科文档 → 代码主线**，供做交接和验收的人逐项核对。

> 本表只做对照与定位。真实凭据/服务器信息按协议第 4.3 款单独编制、走凭据渠道移交，不写入本百科。

## 一、开发交接文档（协议 4.2）↔ 本百科系统篇

| # | 协议里的模块 | 对应仓库 | 本百科文档 |
|---|---|---|---|
| 1 | 微信服务平台 | wxdevops-task-admin, bestfunc_wechat_plugin | [systems/01-wechat-service.md](../systems/01-wechat-service.md) |
| 2 | 数据同步服务 | sync-data（服务端） | [systems/02-data-sync-service.md](../systems/02-data-sync-service.md) |
| 3 | 数据同步客户端 | sync-data（sync-client） | [systems/03-data-sync-client.md](../systems/03-data-sync-client.md) |
| 4 | SmartQuality APP V2 安卓端 | smart_tpm_app（安卓）；SmartQuality-client（桌面） | [systems/04-app-client.md](../systems/04-app-client.md) |
| 5 | SmartQuality APP V2 Web 端 | smart_tpm_web_v2 | [systems/05-web-platform.md](../systems/05-web-platform.md) |
| 6 | SmartQuality 移动套装 | smart_quality_app_v2 / Tinia_nodes_smart_quality_app / smart_quality_plugins / framework_v3_smart_quality | [systems/06-mobile-suite.md](../systems/06-mobile-suite.md) |

> 每篇系统文档都含协议要求的"整体架构/技术栈/目录结构/核心逻辑/配置项/已知缺陷/未完成工作/从哪读起代码"，即开发交接文档正文。

## 二、代码仓库（协议 4.1）↔ 实际主线分支

> 协议 4.1.1 要求推送至"主干或指定分支"。下表"活跃主线"按"远程各分支提交最多最新者"判定，接手/验收时以此为准，避免看错分支。

| 仓库 | 托管 | 活跃主线（提交最多） | 备注 |
|---|---|---|---|
| smart_tpm_web_v2 | 云效 | **dev**（master 仅 15 提交且旧） | Web 端主线在 dev |
| sync-data (bestfunc-data-sync-server) | 云效 | **main** | 客户端+服务端同仓 |
| smart_quality_app_v2 (SmartQuality-client) | 云效 | **main** | 桌面采集端（Electron） |
| smart_tpm_app | 云效 | **feature/app-v2**（master 停在一年前） | 安卓采集端（Flutter） |
| bestfunc-wechat (wxdevops-task-admin) | 云效 | main | — |
| bi | 云效 | **dev**（master 较旧） | — |
| framework_v3_smart_quality | GitHub | main = dev（同点） | 日报平台 app 源码 |
| bestfunc_wechat_plugin | GitHub | main | — |
| Tinia_nodes_smart_quality_app | GitHub | main | — |
| smart_quality_plugins | GitHub | — | 插件 marketplace（本百科所在仓库） |

## 三、部署运维交接（协议 4.3）——不在本百科

协议 4.3 的部署信息、服务器清单、账号凭据按第 4.3.4/4.3.5 款**单独编制、单独交付甲方留存，不作为附件、不写入本百科**。本百科各系统篇只描述"部署形态与启动方式"的通用结构（docker-compose/systemd/launchd/electron-builder 等），不含具体服务器地址与凭据。

## 四、验收要点速查（协议第五条）

对照协议 5.2 验收合格标准，用本百科辅助核对：

| 验收标准 | 用本百科哪里辅助 |
|---|---|
| 代码可在指定环境编译/构建/运行 | 各 [systems/](../systems/) 篇的"技术栈/从哪读起"+ [reference/09-tech-stack.md](../reference/09-tech-stack.md) |
| 交接文档完整、与生产实际一致 | 各系统篇正文 + [faq/01](./01-successor-questions.md) 的"已知文档待校对" |
| 账号凭据经实际登录验证可用 | 走凭据渠道（不在本百科） |
| 接手人能独立运维/排障/开发 | [scenarios/01-successor-onboarding.md](../scenarios/01-successor-onboarding.md) + 各系统"日志与故障排查" |

> 提示：本百科可作为交接"开发文档"部分的载体或索引，但**不替代**协议要求单独交付的部署信息清单与凭据清单。

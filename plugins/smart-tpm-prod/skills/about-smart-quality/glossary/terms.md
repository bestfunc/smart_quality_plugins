# 术语表（Glossary）

SmartQuality（内部亦称 Smart TPM）产品全域术语与缩写。首次在其他文档出现的专有名词都收录于此。按域分组，域内大致按"先出现先讲"。

> 跨文档统一口径：**SmartQuality = Smart TPM**，同一产品套装的两个内部叫法。

---

## 一、产品总体

| 术语 | 解释 |
|---|---|
| **SmartQuality / Smart TPM** | 同一产品套装的两个内部叫法。TPM=Total Productive Maintenance（全员生产维护），历史项目代号 |
| **产线声学质量检测** | 产品核心场景：对产线上产品录音，用算法判合格与否，替代人工听感质检 |
| **OK / NG** | 质检结论。OK=合格；NG（No Good）=不合格 |
| **移动套装** | "人拎着走"的便携录音采集组合：便携硬件 + 桌面/手持采集 App + 上云与 AI 分析链路 |
| **PCM 音频** | Pulse Code Modulation，最原始的无压缩数字音频格式，录音的原始形态 |
| **字节序（byte-order / byte-swap）** | 多字节数值在内存中的排列顺序（大端 BE / 小端 LE）；音频弄反会变噪声 |
| **FFT / FFTW / fftea** | 快速傅里叶变换（把音频转频谱做声学分析）及其两种实现库（C 版 FFTW / Dart 版 fftea） |

## 二、检测平台域概念

> 详见 [reference/04-key-concepts.md](../reference/04-key-concepts.md)。层级：company → project → product → workstation → detect_point → detect_stage → detect_flow。

| 术语 | 解释 |
|---|---|
| **组织 company** | 租户单位，数据按 `companyId` 硬隔离的最高边界 |
| **项目 project / 产品 product** | 一次交付/一条产线的业务容器 / 其下被检测的具体产品型号 |
| **工位 workstation** | 物理产线作业位；用户可见性按工位组装 |
| **检测点 detect_point / 阶段 detect_stage** | 工位上"在哪儿测"的物理位置 / 检测点内"第几步测"的时序步骤（一阶段绑一个检测流程） |
| **检测流程 detect_flow** | 一个阶段实际跑的算法工作流图，版本化（detect_flow_version），平台核心配置对象 |
| **流程节点 detect_flow_node** | 流程图里的节点，类型 algorithm/merge/branch/output |
| **流程执行 detect_flow_execution** | 检测流程跑一次的执行实例 |
| **算法 algorithm / 算法流 algorithm_flow / 算法参数 algorithm_param** | 单个算法注册条目 / 多算法编排的子工作流 / 可复用的参数模板 |
| **BF_\* 算法** | 声学后端算法命名前缀（分类/评分/特征/异常/脉冲/电机/分贝等） |
| **数据集 dataset / 样本 sample（dataset_item）** | 标注样本集合 / 集合里的一条数据（媒体+标注） |
| **粗标 rough / 细标 detail** | 样本级整体判定（OK/NG/unknown）/ ROI（感兴趣区域）级框选标注 |
| **标注 schema annotationSchema / 标注状态 markCode** | 定义可打哪些 label 的模板 / 标注完成度状态机（unmarked→coarse_done→fine_done→locked） |
| **抽样比例 sampleRatio** | 0~1，从数据集抽多少比例进测试任务 |
| **测试任务 test_task / 测试记录 test_record** | 批量跑检测流程的离线批次 / 其中单条样本的执行结果 |
| **生产检测记录 detect_record** | 产线实时产生的检测记录，按工位分库存放 |
| **工位分库** | 生产记录按工位/通道维度分库分表的工程约束 |
| **检测通道 detect_channel（channel_id）** | 生产记录挂载的通道维度，链路追踪入口 |
| **链路追踪 trace** | 从记录反查"流程执行→阶段→节点算法"的三层链路 |
| **标注审计 DetectAnnotationHistory** | 记录每次标注写操作的审计表 |
| **联网录音 net_audio / SmartAudio** | 工业现场联网长时录音系统，按时间戳回放、麦克风 A/B/C/D 四通道。子百科 [about-net-audio](../../about-net-audio/SKILL.md) |
| **音频质检后端 audio-quality / ai_api_server_v2** | DAG 引擎驱动、跑 BF_* 声学算法的算力后端。子百科 [about-audio-quality](../../about-audio-quality/SKILL.md) |

## 三、数据同步（sync-data）

> 详见 [systems/02](../systems/02-data-sync-service.md) / [systems/03](../systems/03-data-sync-client.md)。

| 术语 | 解释 |
|---|---|
| **块（block）** | 同步的最小传输兼事务单位：一张根表若干行 + 关联子表 + 文件引用，打包成 JSON |
| **源端 / 目标库** | 源端=产线本地业务 MySQL；目标库=中心侧按客户端隔离的汇聚库 |
| **Binlog** | MySQL 记录所有数据变更的二进制日志 |
| **现场 site / client_id** | 一套独立部署的采集端实例 / 其全局唯一标识，服务端据此隔离目标库 |
| **mode（default / sqMode）** | 客户端运行模式：完整同步管道 / 只跑本地上传队列不依赖源端 |
| **h2c** | HTTP/2 明文（无 TLS）传输，gRPC 走这条便于经 Nginx 分流 |
| **UPSERT / INSERT IGNORE** | 撞主键时"覆盖全字段"/"静默跳过"两种幂等写入 |
| **ReconcileSchema（列对账）** | 解析源表 DDL 只补目标表缺失列，绝不改类型/删列 |
| **三层去重** | 块级（block_id）+ 文件级（比大小跳过）+ 行级（UPSERT）三粒度防重复 |
| **能力协商 / CommitBlock** | 服务端灰度放开能力后客户端才启用新路径 / v3.4 音频先分片落 MinIO 再原子提交块 |
| **隔离 quarantine / 死信 dead_letter / unquarantine** | 多次失败的块进隔离区 / 兜底死信 / 释放重发的运维工具 |
| **反压 backpressure / 断路器 circuit breaker** | 下游写不动时的分级流控 / 目标库异常时自动熔断保护 |

## 四、采集 App（客户端）

> 详见 [systems/04](../systems/04-app-client.md)、[systems/06](../systems/06-mobile-suite.md)。

| 术语 | 解释 |
|---|---|
| **smart_tpm_app** | **安卓/手持采集端**本体，Flutter/Dart 编写，产出 Android APK（实际主线在 `feature/app-v2` 分支） |
| **SmartQuality-client（smart_quality_app_v2）** | **桌面采集端**，Electron + React（`electron-builder.yml` 佐证），Windows 为主，不打安卓包 |
| **Flutter / Dart · Electron · Capacitor** | 安卓端框架 / 桌面端框架 / 未使用（用于排除桌面端是安卓的可能） |
| **商米 Sunmi（SunmiOpenService）** | 安卓工业手持终端及其原生 SDK |
| **SV30（声学盒子 / box server）** | 现场声学采集硬件，HTTP + WebSocket 出音频流 |
| **appapi v2 / edge_api** | 安卓端对接的新边缘后端接口族及其两代后端 |
| **Modbus-TCP / PLC** | 安卓端经 Modbus 轮询 PLC，自动触发检测与条码转发 |
| **Riverpod / freezed / go_router** | 安卓端状态管理 / 不可变模型代码生成 / 声明式路由 |
| **SeaweedFS** | 桌面端内嵌的对象存储 |
| **keystore（密钥库）** | 安卓正式签名所需（当前 release 仍用 debug 签名，为已知待整改项） |

## 五、微信服务平台

> 详见 [systems/01](../systems/01-wechat-service.md)。

| 术语 | 解释 |
|---|---|
| **WeCom（企业微信）** | 腾讯面向企业的办公 IM，本平台唯一接入的微信形态（无公众号/小程序） |
| **会话内容存档 / wecom_finance_sdk** | 企业微信官方合规留痕能力 / 拉取并解密存档消息的原生 SDK |
| **云效** | 阿里云研发协作平台，AI 识别出的任务指派在此建工作项（当前主线被注释禁用） |
| **邮箱即唯一身份** | 平台身份以邮箱为准，不与微信绑定；RBAC、群授权、MCP 可见范围均按邮箱判定 |
| **can_view / can_manage** | 两套权限：MCP 只读可见的群 / Web 后台可编辑指派的群 |
| **access_level（密级）/ biz_category** | 群的四级密级（public/internal/confidential/secret）/ 内外部标记 |
| **@提及回调** | 群消息 @ 到指定人时按规则用 HMAC 签名推送到外部 Webhook |

## 六、AI 插件层与生态

> 详见 [systems/06](../systems/06-mobile-suite.md)、[reference/12](../reference/12-ecosystem.md)。

| 术语 | 解释 |
|---|---|
| **MCP（Model Context Protocol）** | 把后端能力以标准工具形式暴露给本地 AI 客户端的协议 |
| **scope（作用域）** | MCP 权限按"模块×读/写"切分的授权粒度，如 `mcp:datasets:read` |
| **OAuth 2.1 + PKCE + DCR** | 插件免 token 的浏览器一次同意授权（授权码 + 防截获校验 + 客户端动态注册） |
| **ApiKey 鉴权** | 服务到服务中转的鉴权方式，不绑用户（区别于 OAuth） |
| **marketplace / variant（local/test/prod）** | 一个 Claude Code 插件仓库 / 同内容不同后端环境的三个 plugin 变体 |
| **`_shared/skills`** | 所有 skill 的唯一原件来源（single source of truth），发版物理复制到各变体 |
| **百科型 vs 业务型 skill** | user-invocable 纯文档知识 / Reference 速查或 Task 干活 |
| **子百科** | 各讲一块细分层的 about-\* skill（本百科讲全貌，子百科讲细分层） |
| **姊妹插件 SmartTPM_Files_Plugin** | 同一 MCP 端点的文件下载插件 |
| **Tinia / Tinia blob** | 内部拖节点连 DAG 的分析/日报平台 / 其二进制文件存储区 |
| **table_prefix / handler** | 插件专用 DB 表统一前缀（如 `plg_smart_quality_`）/ 插件里被平台按需调起的一段代码 |
| **凭证 / 数据源** | "连某系统的账号信息(host+token)" / "用哪个凭证、默认拉哪个项目"的配置 |
| **daily / plugins / apps 三类 app** | Tinia v3 上的三种进程模型：日报(请求跑一次) / 节点插件(one-shot) / 常驻服务(always-on) |
| **活报告 / 静态快照** | 每次打开实时重跑取数的分享页 / 用无头浏览器烤死的永久 HTML |
| **slug / alias** | 决定 app URL 的小写连字符目录名 / 凭据引用名（下划线命名，不能连字符） |

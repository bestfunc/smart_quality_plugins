# SmartQuality 移动套装 / AI 分析链路

> 交接文档模块 · 系统篇 06
> 覆盖对象:**移动套装**这套"软硬件 + 上云 + AI 分析"组合，以及把它接进 AI 的三个仓库
> `Tinia_nodes_smart_quality_app` / `framework_v3_smart_quality` / `smart_quality_plugins`。

---

## 一、这是什么(概览)

**SmartQuality（内部亦称 Smart TPM）** 是产线声学质量检测平台。它有两条"入口"采集现场声音：一条是固定安装在产线上的**联网录音系统**（`net_audio` / SmartAudio，嵌入式盒子，详见姊妹百科 `../../about-net-audio/SKILL.md`）；另一条就是本文主角——**移动套装**，一套让人拿着走、在任意工位快速录音打样的**便携组合**。

**移动套装 = 便携硬件 + 一个桌面 App + 上云与 AI 分析链路**。它的定位是"哪里要测就拎到哪里"：不依赖产线预埋的固定录音盒，工程师带着套装到现场，对着待测件录一段声音，当场判 OK/NG，需要时再把录音同步上云、喂给算法与 AI 复核。

### 套装由哪些件组成

| 件 | 形态 | 作用 | 术语解释 |
|---|---|---|---|
| **便携采集硬件** | 麦克风 / 声学采集头 + 便携主机（笔记本或工控机） | 现场对待测件录制原始声音（PCM 波形） | **PCM**＝脉冲编码调制，未压缩的原始数字音频采样 |
| **SmartQuality-client** | 桌面 App（Wails 打包，内嵌 Express 后端 + React 前端） | 现场录音、试听、当场判定、按项目/工位/条码归档，并向云端同步 | **Wails**＝把 Go 后端 + Web 前端打成单机桌面 App 的框架；内嵌 **Express**＝Node.js 的 HTTP 服务，App 本地监听（默认 `http://localhost:3001`） |
| **本地存储 / 对象存储** | SeaweedFS 等 | 存放录音 WAV/PCM 文件，按 fileKey 取用 | **SeaweedFS**＝一个分布式对象/文件存储；录音以字节流形式存在其中 |
| **上云与账号** | REST API + Token 鉴权 | 把 client 的项目、录音、元数据对外提供，供分析侧按需拉取 | **REST API**＝基于 HTTP 的数据接口；**Token**＝一段代表"已登录用户"的令牌 |

> SmartQuality-client 的源码不在本文调研的三个仓库里，本篇只从"它对外暴露了什么"（REST API、`/plugin-auth` 授权页、`?format=wav` 下载）来描述它。要写 client 本体，请回 SmartQuality-client 仓库。

### 采集 → 分析 → AI 的链路

```
 现场待测件
     │ 录音
     ▼
┌─────────────────────┐
│  便携采集硬件         │  麦克风 / 声学头 + 便携主机
└─────────────────────┘
     │ PCM 波形
     ▼
┌─────────────────────────────────────────┐
│  SmartQuality-client（桌面 App）          │  现场判 OK/NG、按项目/工位/条码归档
│   · 内嵌 Express，本地 :3001              │  录音存对象存储（SeaweedFS）
│   · 对外 REST API + Token 鉴权            │  /plugin-auth 授权页、/storage/download?format=wav
└─────────────────────────────────────────┘
     │ REST 拉取（不直连 DB）
     ▼
┌─────────────────────────────────────────┐
│  Tinia_nodes_smart_quality_app（插件）    │  把 client 录音接成 Tinia「数据集节点」
│   · 凭证（Token）+ 数据源 + 多维筛选       │  可选并行下 WAV 到 Tinia blob
└─────────────────────────────────────────┘
     │ Dataset（带条码/产品/工位/状态等元数据）
     ▼
┌─────────────────────────────────────────┐
│  Tinia / framework_v3_smart_quality       │  DAG 分析流水线 · 日报平台 App
│   · daily 日报 / plugins 节点 / apps 常驻  │  算法节点算频谱/声级、出 AI 质检日报
└─────────────────────────────────────────┘
     │ 检测记录 / 标注 / 日报数据落 DB
     ▼
┌─────────────────────────────────────────┐
│  smart_quality_plugins（本仓库）           │  MCP 把 9 模块 61 tool 暴露给本地 AI
│   · Claude Code / Codex / Qwen Code        │  自然语言查数据、追链路、辅助打标
└─────────────────────────────────────────┘
     │
     ▼
   本地 AI（工程师用自然语言问 / 复核 / 打标）
```

一句话：**现场录音 → SmartQuality-client 归档并上云 → Tinia_nodes 把录音接成数据集 → Tinia/framework_v3 跑分析与出日报 → smart_quality_plugins 用 MCP 把结果交到本地 AI 手里。** 三个仓库分别负责链路的"接入""分析""AI 暴露"三段。

---

## 二、三个仓库怎么协同(主体)

### 2.1 `Tinia_nodes_smart_quality_app` —— 把录音接进 Tinia 的插件

**Tinia** 是一套可视化数据分析平台：你在画布上拖若干**节点**连成一条流程（DAG，有向无环图），数据从上游节点流到下游。本仓库是一个 **Tinia 插件**——把 SmartQuality-client 的录音变成 Tinia 里可用的**数据集节点（Dataset 节点）**，让录音能进入 Tinia 的分析流。

**关键设计：只走 REST，不碰库。** 插件通过 SmartQuality REST API 拉数据（`/projects`、`/recordings/groups`、`/recordings/upload-list`、`/storage/download`），不直连 client 的数据库。这样 client 与分析侧解耦，client 改库不影响插件。

**插件机制与节点定义**（清单文件 `tinia-repo.yaml`）：

| 概念 | 值 / 说明 | 术语解释 |
|---|---|---|
| **`table_prefix`** | `plg_smart_quality_` | 插件专用数据库表的统一前缀，避免跨插件撞表（本插件建 `plg_smart_quality_credentials` 存凭证、`plg_smart_quality_datasources` 存数据源） |
| **migrations** | `migrations/001_init.up.sql` | Tinia 启动时按文件名排序自动执行的建表脚本，跳过已执行的 |
| **credentials（凭证）** | `smart_quality_basic`：host + token 两字段 | 凭证＝一份"连某个 client 的账号信息"。**只存 host 与 token，邮箱密码不入库**；UI 严格只增删、无编辑（token 失效就删旧建新） |
| **datasources（数据源）** | `smart_quality`：绑一个凭证 + 默认项目 | 数据源＝"用哪个凭证、默认拉哪个项目"的一份配置 |
| **nodes（节点）** | `smart_quality_dataset`（category: source，输出 `Dataset`） | 核心数据集节点；`node.yaml` 声明 `outputs`、`params_schema`、`runtime` |
| **handlers** | `test_connection.py` / `fetch.py` | **handler**＝插件里被平台按需调起的一段 Python：连接测试（`GET /auth/me` 验 token）与数据拉取（stdin 收 `{credential, config, query}`，stdout 吐 `{items, total, page}`） |

**节点筛选字段**（`schemas/params.schema.json` → 属性面板表单）：

| 字段 | 类型 | 服务端还是客户端过滤 | 说明 |
|---|---|---|---|
| 项目 | 下拉 / 文本 | 服务端 | 必选；从 client `/projects` 拉；留空用数据源默认项目 |
| 关键词 | contains | 服务端 | 条码 / 产品名模糊匹配；填了走 `recordings/groups` 接口 |
| 开始 / 结束日期 | `YYYY-MM-DD` | 服务端 | 同上走 groups 检索 |
| 录音状态 | OK / NG / 全部 | **客户端** | client 无 status 过滤，节点自己判 |
| 产品 / 工位 / 设备 / 阶段 / 工况 / 点位 | contains | **客户端** | 按录音 attribute 逐条匹配 |
| 时长范围（秒） | min / max | **客户端** | 按 `durationMs` 比较 |
| 隐藏已上传 | 勾选 | 服务端 | client `hideUploaded` 参数 |
| 数量上限 limit | 整数 | — | 客户端过滤后的最终条数 |
| 下载音频文件 | 勾选 | — | 开→并行下 WAV 传 blob；关→仅留 `content_url` |

**拉取逻辑**（`handlers/fetch.py`，stdin/stdout 协议）：

```
stdin :  {"credential": {"host","token"},
          "config":     {"default_project_id"},
          "query":      {"project_id","keyword","start_date","end_date",
                         "status":"OK|NG|","hide_uploaded","limit","page"}}
stdout:  {"items":[{"id","name","media_type","content_url","attributes":{...}}],
          "total", "page"}
```

策略：凭证里直接拿 token（不再 login）；有关键词/日期走 `/recordings/groups`，否则走更轻量的 `/recordings/upload-list`（含上传/导出状态）；`project_id` 都没有时只返回 projects 列表供 UI 选。勾"下载音频文件"时，节点并行下载 WAV 到 **Tinia blob**（Tinia 的二进制文件存储区，落在 `~/Tinia/blobs/`）。

**凭证存储**：`plg_smart_quality_credentials.data`（JSON 文本）只持久化 `{"host":"http://localhost:3001","token":"<Bearer JWT>"}`——**邮箱和密码不入库**，授权页签发完 token 后账号密码只留在浏览器本地，不回传 Tinia。

**第一次接入流程**（README「使用流程」）：

```
1. 凭证-移动套装 → 新建：名称任意 + 服务器地址(:3001) + 点"打开移动套装授权页"
   → URL 复制到剪贴板(Wails 拦了 window.open)→ 系统浏览器打开授权页填邮箱密码
   → 允许并签发 Token → 复制 token 回 Tinia 粘到"访问令牌" → 保存
2. 数据源-移动套装 → 新建：名称 + 选刚建的凭证 + 点"从移动套装加载"选默认项目
3. 分析流程编辑器 → 拖「移动套装 数据集」节点 → 配数据源+项目+筛选
   → 拖「数据下载(Materialize)」节点连线 → 跑流程 → WAV 落 ~/Tinia/blobs/
```

其中 **Materialize 节点**＝Tinia 内置的"把引用型数据实体化落盘"节点，负责把数据集里的 `content_url` 真正下载成本地文件。

> 授权流：client 一侧提供独立 HTML 授权页 `/plugin-auth/:app`（不依赖其 React 前端），`authMiddleware` 支持 `?token=` query 兜底（让 Tinia 的 materialize 裸 HTTP GET 也能过鉴权），`/storage/download/:fileKey?format=wav` 会拼 44 字节 RIFF/WAVE 头并做字节序转换，返回任何播放器可播的合法 WAV。**字节序**（大端 BE / 小端 LE）＝多字节数值在内存里的排列顺序，弄反声音会变噪声。

**client 侧配套改动**（本插件依赖 SmartQuality-client 仓库这几处，需在 client 单独 PR/commit）：新增 `/plugin-auth/:app` 授权页路由（Express 直接 serve）；`authMiddleware` 支持 `?token=` query 兜底；`storage/download/:fileKey` 加 `?format=wav`（拼 WAV 头 + BE→LE byte-swap）；开发期 CORS 允许 Wails WebView origin。也就是说，"接得通"不只是插件单方能力，还需要 client 端开这几个口子。

**数据在链路中的形态变化**：

| 阶段 | 数据形态 | 承载者 |
|---|---|---|
| 现场采集 | 原始 PCM 波形 | 便携硬件 |
| client 归档 | 录音 + 元数据（条码/产品/工位/状态/时长…），文件存对象存储 | SmartQuality-client |
| 插件接入 | Tinia `Dataset`（`items[]` 带 `content_url` + `attributes`） | Tinia_nodes 数据集节点 |
| 分析产出 | 频谱/声级等指标、检测记录、标注、日报 JSON | framework_v3 三类 App |
| AI 暴露 | MCP tool 的结构化返回（`structuredContent`） | smart_quality_plugins |

### 2.2 `framework_v3_smart_quality` —— 日报平台（Tinia v3）的 App 源码仓

这个仓库**只放 App 源码，不放平台代码**。平台是 Tinia 的 v3 形态（内部又叫"日报平台"）；本仓库按 App 类型分三个顶级目录，**版本走 git tag**（`<slug>/vX.Y.Z`，如 `quality-daily-v6/v6.0.11`），不再用文件夹版本。**slug**＝小写连字符的目录名，同时决定 App 的公开 URL `/a/<slug>`。

**三类 App 对比**：

| 目录 | 类型 | 进程模型 | 干什么 | manifest | 入口契约 | SDK |
|---|---|---|---|---|---|---|
| **`daily/`** | 日报（endpoint 型） | 每次请求跑一次（子进程） | 打开链接看一张**活报告**，或定时烤一张**静态快照** | `config/plugin.yaml` | `def fn(params) -> dict`，return 即响应 | `from report_sdk import sdk` |
| **`plugins/`** | 节点插件（dag / v3 型） | one-shot worker（Pool 调度） | 字典维护 / 数据加工 / DAG 流水线，可带自定义 UI 页 | `tinia-repo.yaml`（`modules.nodes`） | `Runtime.from_stdin()`，stdin→stdout 逐行事件 | `from bestfunc_sdk import Runtime` |
| **`apps/`** | 常驻服务（service 型） | 常驻（Supervisor 托管，退避自愈） | 实时数据流 / 长连接桥接 / always-on | `tinia-repo.yaml`（`service:` 块） | `def serve(ctx)` | `report_sdk.service_main` 拉起，用 `ctx` |

术语：**活报告**＝页面用 `window.dr.call('<endpoint>')` 实时拉数据渲染；**静态快照**＝把某一刻的数据烤进 HTML（`snap.create()`），读 `window.__SNAPSHOT__`，不再请求后端；**KV**＝App 级键值存储（`sdk.kv`，64KB/key）；**endpoint**＝一个可被前端调的后端函数。

**三类 App 之间的关系**：三者不是层级关系，而是**同一平台上按运行形态分工的三种 App**，围绕同一批检测数据协作——

- `daily/quality-daily-v6`：**AI 质检日报**，一份 bundle 部署到所有客户（读老库 `smart_quality_v2`，产线名/工位名从 DB 动态读），每天 cron 或 UI 触发，出 7 天趋势 + KPI + NG 详情，末尾用 DeepSeek 生成中文总结。
- `plugins/tpm-*`（`tpm-dataset` / `tpm-detect-flow` / `tpm-detect-record` / `tpm-algorithm` / `tpm-model-artifact` / `smart-tpm-*` 等）：把 Smart TPM 各业务模块做成 v3 节点插件（字典维护 / 数据加工），是 MCP 之外的一条"平台内可视化操作"路径。
- `apps/tpm-realtime-app` / `sv30-relay` / `detect-station-*`：常驻服务，从联网录音系统实时拉 4 通道音频流、切窗、调 Tinia `run_node` 算频谱/声级，本地 WebSocket 推浏览器实时滚动并转推直播间。

移动套装的录音，经 2.1 的插件接成 Dataset 后，就能进这套 v3 平台被这三类 App 消费（离线批量分析走 daily/plugins，实时监看走 apps）。

**各类 App 的 SDK 落点**（写 App 时最常用）：

| 类型 | 拿参数 | 存数据 | 日志 | 出报告 / 通道 |
|---|---|---|---|---|
| daily | `sdk.params.get(k, default)` | `sdk.kv` / `sdk.daily.read/write` | `sdk.log.info/warn/error`（唯一推荐输出） | `snap.create(template, data, ...)` 烤静态快照；活报告用 `window.dr.call` |
| plugins | `rt.params`（`Runtime.from_stdin()`） | `rt.datasource` 拿工位 DSN 包 | SDK 日志走 stderr | `emit_output`（key 须在 `node.yaml` 的 `outputs` 声明，否则丢弃） |
| apps | `ctx.config`（= manifest 的 `service:` 块） | 需要时 `from report_sdk import sdk` | 一律 stderr | 绑 `ctx.ws_port` 起本地 WS，平台反代 `/rtstream/<slug>`；循环里 `while not ctx.should_stop()` |

**发版 SOP**（README 第 4 节，三类通用）：在 `<类型>/<slug>/` 直接改代码 → 平台 `update_app` / `create_app`（`files` 全量列全该 App 所有文件）→ 轮询 `get_app` 直到 `build_status="ready"` → `try_run_endpoint` / 跑节点 / 看常驻日志验证 → **`publish_app(slug, version_id)` 才真正上线** → 更新 `CHANGELOG.md` → `git commit` + `git tag <slug>/vX.Y.Z` + push。跨环境搬运用平台 `export_app` 拿 bundle、目标环境 `import_app(on_conflict=update)` + `publish_app`，或用平台「插件同步」从本 git 仓直接拉。

**三类通用红线**（写 App 前须知，摘自各 `_template/README.md`）：凭据 **alias** 必须匹配 `^[A-Za-z_][A-Za-z0-9_]{0,63}$`（下划线，不能连字符/数字开头/中文，否则 env 注入失败，且会因平台"精确名不中→兜底最近 verified 同 type"而误绑到别的凭据）；打印中文一律走 SDK 日志（stderr），**绝不 `print` 到 stdout**（会触发 PG `22021 invalid byte`，且 service 的 stdout 是握手协议通道）；`requirements.txt` 必须存在且列全非 stdlib 依赖（空文件会让 builder 短路成功却 libDir 空、import 就炸）；**save ≠ publish**（`update_app` 只建版本，`publish_app` 才上线，且 `files` 是全量替换、漏一个丢一个）；endpoint 5 分钟连崩 3 次会被平台自动下架（改完先看 traceback 再 publish）。

### 2.3 `smart_quality_plugins` —— MCP 把数据交给本地 AI（本百科所在仓库）

这是一个 **Claude Code 插件 marketplace**（插件市场）：通过 **MCP（Model Context Protocol，模型上下文协议，让 AI 客户端调用外部工具的标准协议）**，把 Smart TPM 后端的 **9 个模块、61 个业务 tool** 暴露给本地 AI 客户端（Claude Code / Codex / Qwen Code），让工程师用自然语言查检测数据、追溯算法链路、复现客户报错、辅助打标记。**本 about-smart-quality 百科就住在这个仓库**（`plugins/_shared/skills/`）。

**定位**：它是链路最末端的"AI 接入层"——不做采集、不做分析，只把前面沉淀下来的数据（数据集、检测流程、测试任务、算法、生产检测记录等）以 tool 形式交到 AI 手里。三环境变体：`smart-tpm-prod` / `smart-tpm-test` / `smart-tpm-local`，切环境改 `.claude/settings.json` 的 `enabledPlugins` 后缀再重启。鉴权走 **OAuth 2.1 + PKCE + DCR**（授权码 + 防截获校验 + 客户端动态注册；浏览器一次同意即可，不用复制粘贴 token），按"模块 × 读写"切分为 7 个 **scope**（权限范围），其中 `detect_flows` / `algorithms` 不开写权限以防 AI 改坏生产配置。

**9 模块 61 tool 一览**（读为主，写受 scope 严控）：

| 模块 | tool 数 | 代表 tool |
|---|---|---|
| 数据集 & 标注 | 7 | `datasets_query_samples`、`datasets_export_manifest`（一次拿全量清单） |
| 标记系统（读+写） | 9 | 读 `project_marks_list`；写 `marks_set_rough` / `marks_add_detail`（需 `mcp:datasets:write`） |
| 检测流程 | 5 | `detect_flows_list/get/list_nodes`、`algorithm_flows_*` |
| 算法 | 5 | `algorithms_list/get`、`algorithm_params_search` |
| 测试任务 & 记录 | 5 | `test_tasks_*`、`test_records_*` |
| 生产检测记录 | 3 | `detect_records_list/get/ping`（工位分库） |
| 链路追踪 | 9 | `detect_records_trace`（一键拿 3 层链路）+ `detect_logs_*` |
| 元数据反查 | 13 | `products`、`devices`、`assembly_lines`、`workstations_list`、`model_artifacts_*` |
| 配置查询 + 杂项 | 5 | `station_channel_list`、`audio_ai_req_logs_list`（算法引擎调用快照） |

文件下载（对象存储 bucket / 数据集音频 / 模型 artifact）走配套姊妹插件 `SmartTPM_Files_Plugin`——两插件指向同一 MCP server，OAuth 一次浏览器同意两个都生效。写操作（`marks_*`）自动写审计表、`createBy/updateBy` 用被授权真实用户 id，可用 `annotation_history_list` 反查"AI 哪次会话改了什么"。

**已有的 about-* 子百科导航**（都与本 `about-smart-quality` 同级，在 `_shared/skills/` 下，用相对路径互链）：

| 子百科 | 讲什么 | 相对路径 |
|---|---|---|
| `about-smart-tpm-mcp` | 插件 / MCP 集成层：marketplace 结构、3 变体、OAuth、7 scope、61 tool、13 个 skill 导览、姊妹文件插件 | [`../../about-smart-tpm-mcp/SKILL.md`](../../about-smart-tpm-mcp/SKILL.md) |
| `about-audio-quality` | 音频质检后端推理引擎（`ai_api_server_v2`）：DAG 流程引擎、检测引擎与双视角聚合、21+ 算法模型、GPU 集群 | [`../../about-audio-quality/SKILL.md`](../../about-audio-quality/SKILL.md) |
| `about-net-audio` | 联网录音系统（SmartAudio）：双进程架构、A/B/C/D 通道、灵敏度调校、算法直调、现场部署 | [`../../about-net-audio/SKILL.md`](../../about-net-audio/SKILL.md) |
| `about-sync-data` | 数据同步链路（SmartQuality-client → 云端 / 平台的上传与对齐） | [`../../about-sync-data/SKILL.md`](../../about-sync-data/SKILL.md) （截至本篇撰写，marketplace 内**尚未见**该 skill 目录；如你接手时它已建，本链接生效，否则以移动套装的同步描述与 client 仓库为准） |

> 分工记忆法：`about-net-audio` 讲**固定录音前端**、本篇讲**移动录音套装**、`about-audio-quality` 讲**算法后端**、`about-smart-tpm-mcp` 讲**AI 接入层**、`about-sync-data`（若有）讲**同步管道**。

**本篇边界**：这里讲的是"移动套装如何一路接到 AI"这条端到端链路，聚焦三个仓库的协同与接缝。三个仓库各自的完整实现细节（client 本体的现场判定逻辑、Tinia 平台内核、每个 tool 的入参出参、每个 App 的业务字段）不在本篇展开，请回各自仓库的 README / skill。本篇不写代码、不调用 MCP 工具；要实际操作数据集/检测流程/打标，请用 `smart_quality_plugins` 里的业务 skill。

---

## 三、版本关系、缺陷与从哪读起(收尾)

### 3.1 框架版本关系与升级路径（措辞从弱）

| 仓库 | 版本锚点 | 现状 |
|---|---|---|
| `Tinia_nodes_smart_quality_app` | `tinia-repo.yaml` `version: 1.0.3`，`min_tinia_version: "1.16"` | 依赖 Tinia ≥ 1.16；节点 `smart_quality_dataset` 内部 `version 1.0.0` |
| `framework_v3_smart_quality` | 每个 App 独立 semver，git tag `<slug>/vX.Y.Z`（如 `quality-daily-v6/v6.0.11`） | App 各自演进，平台为 v3 形态；历史版本靠 git tag 回溯 |
| `smart_quality_plugins` | marketplace `v1.0.13`，后端 tool 随配套 web_v2 版本走 | tool 数随后端增减（如 v1.0.10 加 5 个写 tool；文档口径以后端 `tools/list` 为准） |

- **`Tinia_nodes` → Tinia v3**：`tinia-repo.yaml` 内注释提到"授权现走 token 临时输入、后续 client 上线 `/plugin-auth/tinia` 授权页后切到跳转流"，可视作一条渐进的接入方式演进，跳转授权可在 client 侧就绪后逐步替换手动粘贴。
- **`framework_v3`**：README 记载旧仓曾用 `current/ + vX.Y.Z/ 文件夹 + bundle.dr-app.json` 归档，现已全部 git 化；`quality-daily-v6` 取代了手工 copy-paste-rename 的客户专属 v5，往"一份 bundle 通吃多客户"方向收敛。
- **`smart_quality_plugins`**：CHANGELOG 呈现的是"随后端能力增量加 tool / 加 skill"的节奏（读 → 读写 → 文档百科），倾向持续补充而非大改结构。
- 版本对齐建议：写 App 时 `requirements.txt` 尽量对齐平台 `/opt/wheelhouse` 预装的锁版本 wheel（`PyMySQL==1.2.0`、`requests==2.32.3`、`pandas==2.2.3`、`anthropic==0.40.0` 等），有助于客户机离线 build 命中。

### 3.2 已知缺陷 / 后续建议

摘自 `Tinia_nodes` README「已知问题」与各 `_template` 红线：

1. **Wails WebView 拦 `window.open`**：桌面版 WebView 沙箱不暴露浏览器打开 API，"打开授权页"按钮目前靠剪贴板 + 用户手动粘贴。后续 Wails 暴露 `runtime.BrowserOpenURL` 后可去掉这个 workaround。
2. **`materialized_files` 外键警告**：Tinia 内置 materialize 节点想把下载文件登记到主库 datasource 表，插件用的是插件表 ID，导致外键冲突——下载本身成功，仅缓存索引没建。后续可考虑让 materialize 兼容插件表来源。
3. **API 不支持细粒度过滤**：client 的 `upload-list` / `groups` 只支持项目/日期/关键词/上传状态过滤，产品/工位等业务标签全在节点客户端做。这意味着"先按 limit 拉一批再过滤"，筛选严时建议把 limit 调大（如 500），避免过滤完只剩一两条。若 client 侧后续开放服务端标签过滤，可下沉此逻辑。
4. **录音字节序假设**：节点假设对象存储里的 PCM 是大端、需 byte-swap（沿用 client 的 `sq-audio-copier` 处理）；若其它设备录入的是小端 PCM，会被错误 swap。建议随录音落一个显式字节序标记以消歧。
5. **凭证只增删**（产品要求，非缺陷）：token 失效或换账号必须删旧建新，避免半改状态。

### 3.3 接手人从哪读起

1. **先建立全局**：读本篇的 ASCII 链路图与三类 App 对比表，搞清"采集→接入→分析→AI"四段各属哪个仓库。
2. **要接录音进分析**：读 `Tinia_nodes_smart_quality_app/README.md` + `tinia-repo.yaml`，跟着"第一次接入"三步走一遍（建凭证 → 建数据源 → 拖数据集节点 → 拖 materialize 节点跑流程）。
3. **要写/改平台 App**：从 `framework_v3_smart_quality/README.md` 选对 App 类型，再读对应 `daily/_template`、`plugins/_template`、`apps/_template` 的 README（含 SDK 速查与红线）；参考实例 `daily/quality-daily-v6`、`apps/tpm-realtime-app`。
4. **要用 AI 查数据**：读 `smart_quality_plugins/README.md` + `CHANGELOG.md`，装插件、过 OAuth、勾 scope，再看业务 skill（`quickstart-smart-tpm-mcp` 起步）。
5. **要补背景**：按 3.1 表右列跳到 `about-net-audio`（固定录音前端）、`about-audio-quality`（算法后端）、`about-smart-tpm-mcp`（AI 接入层）三个姊妹百科，与本篇的移动套装视角互补。

> 一句话交接：移动套装是"人拎着走的那套录音 + 上云 + AI 复核"，`Tinia_nodes` 负责把它的录音接进分析平台、`framework_v3` 负责分析与出日报、`smart_quality_plugins` 负责把结果交给本地 AI。三段之间的接缝（REST 接口、Token 授权、字节序、全量 files 发布、alias 匹配）是最容易出问题也最值得先摸清的地方。

---

*事实来源：`Tinia_nodes_smart_quality_app`（README + `tinia-repo.yaml` + `handlers/` + `nodes/`）、`framework_v3_smart_quality`（README + `daily/plugins/apps` 三个 `_template/README.md` + `quality-daily-v6` / `tpm-realtime-app` / `sv30-relay`）、`smart_quality_plugins`（README + CHANGELOG + `_shared/skills/` 目录）。数值口径（tool 数、版本号）以对应仓库当次记载为准，随后端演进可能变动。*

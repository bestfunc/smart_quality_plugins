# 数据同步客户端（各产线现场部署）

> 交接文档模块「数据同步客户端」。本文只讲**客户端**（`sync-data/sync-client/`），即部署在**各产线现场**、负责"读源、攒块、上报"的采集端。汇聚端见 [数据同步服务](./02-data-sync-service.md)，整体拓扑见 [架构参考](../reference/02-architecture.md)，名词见 [术语表](../glossary/terms.md)。

## 一句话定义

数据同步客户端是一个 **Go 编写的单文件二进制**，跑在每个产线现场的机器上。它把本地的业务数据（含音频/图片文件）打包成一个个**数据块（block）**，通过 **gRPC** 增量、限速地上报给中心侧的[数据同步服务](./02-data-sync-service.md)。它是「多源端 → 中心库」这条链路的**采集起点（source）**：客户端负责"读源、攒块、发出去"，服务端负责"收块、去重、写库、回填文件 URL"。

- **块（block）**：同步的最小传输单位。分两类——`file_block`（一张根表的若干行 + 关联子表 + 关联文件引用）和 `simple_block`（普通业务表的一批行）。块以 JSON 元数据加可选文件字节流的形式攒在本地磁盘，再由 sender 逐块上报。
- **现场（site）**：一套独立部署的采集端实例，对应一个 `client_id`。每个现场一份自己的 `config.yaml`，互不共享——即「一现场一套配置」（详见第三部分）。

> ⚠️ **给接手人的第一提醒（文档漂移）**：仓库 `README.md` 的技术栈表把客户端本地存储写成 **"SQLite (WAL)"**，`README.md` 项目结构里还列了 `engine/ listener/ parser/ assembler/ buffer/ supervisor/ ui/ web/` 等模块。**当前实际运行的 `sync-client` 二进制并没有这些**：`go.mod` 里根本没有 SQLite 驱动，本地存储是**基于文件系统的块目录 + JSON/YAML 状态文件**（见 2.3）；Binlog 一整套也未在客户端实现，采集靠**周期性增量扫描**。本文只描述**代码能证实**的实现。

---

## 一、客户端架构与运行模式

### 1.1 两种运行模式（`cfg.mode`）

客户端由配置里的 `mode` 字段决定跑哪条采集管道，两者共用同一份下游基础设施（blockstore / sender / heartbeat / 网络探测）：

| 模式 | 数据来源 | 源端依赖 | 典型现场 |
|------|----------|----------|----------|
| `default` | 直连源端 **MySQL** 周期扫描（增量 + 历史 + 简单表），关联文件从源端 **MinIO** 拉取 | 启动期连接 MySQL/MinIO 做自检，连不上即退出 | 有本地业务库、需整库汇聚的现场 |
| `sqMode` | 消费 **SmartQuality-client** 工作站写入本地共享目录的 **jsonl 文件队列** + 音频副本 | **无任何源端依赖**，跳过 MySQL/MinIO 自检 | 只上报质检结果 + 音频的轻量现场 |

`mode` 缺省或非法值一律回退 `default`（`config.go: applyDefaults`）。服务端也能通过心跳响应下发 `mode` 热切换，切换会触发 `main.go` 里的 `OnModeChange` → 停旧模式、启新模式。

### 1.2 进程内组件（`cmd/client/main.go` 装配）

| 组件 | 包 | 职责 |
|------|-----|------|
| **Scanner（default）** | `internal/modes/default` | 增量/历史/简单表扫描源库，产出块入 BlockStore |
| **Scanner（sqMode）** | `internal/modes/sq` | 扫 `pending/` 目录、解析 jsonl、产出 `simple_block`，音频交给上传队列 |
| **BlockStore** | `internal/blockstore` | 本地块文件管理（按日期分目录、去重、隔离、计数器） |
| **Checkpoint** | `internal/checkpoint` | 各表扫描游标持久化（`checkpoint.json`），支持断点续扫 |
| **Sender** | `internal/sender` | 从 BlockStore 按优先级取块，gRPC 上报，带重试/隔离/死信 |
| **ChunkUploader** | `internal/upload`、`internal/modes/sq` | 音频大文件走 MinIO multipart 分片续传 |
| **Heartbeat** | `internal/heartbeat` | 周期心跳：上报状态 + 领取服务端指令/节拍策略/模式 |
| **NetworkMonitor** | `internal/network` | 周期探测服务端在线状态，离线时暂停上报 |
| **PolicyStore** | `internal/policy` | 节拍/限速策略运行时快照 + 本地持久化（`pacing.yaml`/`mode.yaml`） |
| **ResourceLimiter** | `internal/resource` | 内存水位监控（配合 `GOMAXPROCS(1)` + `SetMemoryLimit`） |
| **Status HTTP** | `internal/status` | 本机 HTTP 端点，暴露状态给 SmartQuality-client 与运维 |

> 全进程 `runtime.GOMAXPROCS(1)`、内存上限 `debug.SetMemoryLimit`——客户端刻意压到"单核 + 限内存"，避免抢占现场机的产线软件资源。

### 1.3 数据流（两模式对照）

```
default 模式：
  源端 MySQL ──周期扫描──▶ Scanner ──攒块──▶ BlockStore(本地磁盘)
  源端 MinIO ──拉文件──────┘                      │
                                     Sender ──gRPC 上报──▶ 数据同步服务

sqMode 模式：
  SmartQuality-client ──写 jsonl+音频──▶ sync-export/pending/
                                             │ Scanner 扫描解析
                                             ▼
                                        BlockStore ──Sender──▶ 数据同步服务
                                        音频 ──ChunkUploader 分片──▶ 服务端→MinIO
```

### 1.4 技术栈

| 分类 | 选型 | 备注 |
|------|------|------|
| 语言 | Go 1.24+（`go.mod` 标 1.25） | `CGO_ENABLED=0` 静态单文件，跨 Win/Linux/macOS |
| 通信 | gRPC over HTTP/2 | 双向流上报块 + 一元 RPC 心跳/分片 |
| 本地存储 | 文件系统块目录 + JSON/YAML 状态 | **非 SQLite**（见首部提醒） |
| 对象上传 | MinIO Go SDK（default 模式拉源端文件）、服务端中转 multipart（音频） | |
| 配置 | YAML + `${ENV:default}` 变量替换 | `gopkg.in/yaml.v3` |

---

## 二、对外对接与本地存储

### 2.1 与服务端对接（gRPC）

客户端与服务端共用 `proto/sync.proto` 定义的 `SyncService`，六个 RPC：

| RPC | 类型 | 用途 |
|-----|------|------|
| `SyncBlocks` | 客户端流式 | 老路径：流式发块元数据 + 文件 chunk（32KB/片），服务端回一个 ACK |
| `Heartbeat` | 一元 | 上报状态字段（待发块数、已发字节、隔离计数、扫描进度…），领取指令/策略/模式 |
| `GetUploadProgress` | 一元 | 音频续传：查某 `file_key` 已 ACK 的分片数，断点续传起点 |
| `SendChunk` | 一元 | 上传一个分片（默认 sqMode 8MiB / default 5MiB），服务端转 MinIO multipart |
| `CompleteUpload` | 一元 | 通知分片传完，触发服务端 `complete_multipart_upload` |
| `CommitBlock` | 一元 | v3.4 新路径：音频先分片落 MinIO，再原子提交块元数据（`block_id` 强幂等） |

要点：

- **认证**：每次调用带 `authorization: Bearer <HMAC-SHA256(client_id, secret_key)>` 与 `x-client-id` 元数据（`sender.go: authToken`）。`secret_key` 须与服务端一致。
- **控制面 / 数据面分离（Phase C）**：心跳与网络探测走独立的 `ctrlConn`，上传走 `conn`。历史 bug 是探测连续失败会 `Close` 掉共用连接、腰斩正在上传的块；分离后探测撕连接只影响控制面。
- **块级幂等**：服务端按 `block_id` 去重，重复提交返回 `ALREADY_PROCESSED`，客户端视为成功。
- **能力协商**：服务端心跳广播 `server_capabilities`（如 `commit_block_v1`/`audio_chunked_v1`）；客户端未见到的能力自动 fallback 老 `SyncBlocks` 路径，保证升级兼容。

### 2.2 与产线设备/上位机对接

客户端**不直接连产线 PLC/传感器**。它对接的是上位机侧的软件产物：

- **sqMode 现场**：与本机的 **SmartQuality-client**（见 [App 客户端](./04-app-client.md)）通过**本地共享目录**解耦对接，不走网络。SmartQuality 原子写入 jsonl 与音频副本，sync-client 只读扫描。目录约定（`inbox.go`）：

  | 子目录 | 写入方 | 读取方 | 含义 |
  |--------|--------|--------|------|
  | `sync-export/pending/` | SmartQuality | sync-client | 待处理的 jsonl 队列（`.tmp`/`.` 前缀视为写入中，跳过） |
  | `sync-export/inflight/` | sync-client | sync-client | 取出后立即 mv 到此，防重复处理 |
  | `sync-export/files/` | SmartQuality | sync-client | 音频文件副本，按 workstation 分子目录 |
  | `sync-export/done/`（含 `error/`） | sync-client | 运维 | 成功归档 / 解析失败留证（附 `.err`） |

  根目录 `smartquality_dir` 只填 SmartQuality-client 的 userData 根，四个子目录自动派生（macOS `~/Library/Application Support/SmartQuality`，Windows `%APPDATA%/SmartQuality`）。

- **default 现场**：把"源端本地 MySQL + MinIO"当作上位机数据出口，周期扫描增量。这里没有设备协议，只有数据库轮询。

### 2.3 本地存储与缓冲 / 断点续传

客户端**离线可攒、在线补发**，靠的全是本地文件（数据目录默认 `./data/`）：

| 文件/目录 | 作用 | 断点续传语义 |
|-----------|------|--------------|
| `data/blocks/<YYYY-MM-DD>/` | 待发块（`fb-*` 目录 / `sb-*.json`），按日期分子目录，老格式落 `legacy/` | 进程重启后 sender 自然重扫续发 |
| `data/blocks/.blockstore_state.json` | 计数器（pending/diskBytes）+ 去重索引持久化 | 启动秒级恢复，损坏/负数/漂移时全量重扫自愈 |
| `data/blocks/quarantine/` | 多次重试仍失败的隔离块（不再计入 pending、不再重试） | 运维用 `unquarantine` 工具修因后释放重发 |
| `data/checkpoint.json` | 各表扫描游标（增量/历史/窗口/优先窗口） | 断点续扫，删除即从头全量 |
| `<fb 块>/.inflight.json` | 单块内音频 multipart 上传进度（upload_id/已传分片） | 崩溃后拿 upload_id 调 `GetUploadProgress` 续传 |
| `data/pacing.yaml` / `data/mode.yaml` | 断网期间沿用上次收到的节拍策略 / 模式 | 重启不丢策略 |
| `data/client_id.dat` | 客户端唯一 ID（未在配置固定时自动生成） | **勿删**，删除后服务端视为新客户端 |
| `data/dead_letter/` | 达到重试上限、隔离也失败时的兜底死信块 | 人工排查 `reason.txt` |

缓冲/续传的三条主线：

1. **块缓冲**：网络离线时 sender 停发（`NetworkMonitor.IsServerOnline` gate），块继续攒在 `blocks/`，磁盘水位驱动上游背压。
2. **发送重试与隔离**：sender 对瞬时错误指数退避（±25% jitter）；`PERMANENT` 错误或超 `MaxTransientRetries`（默认 10）即隔离；`HasFailedFiles` 块连续跳过 60 次也强制隔离，避免源文件永久 404 卡死管道。
3. **分片续传**：音频按分片上传，`GetUploadProgress` 返回已 ACK 分片数，客户端从下一片续传，重启不重头。sqMode 上传另有**内存重试队列**（10s/30s/2min 三档退避，离线暂停不消耗次数）——注意此队列进程重启即丢，靠 SmartQuality 端 backfill nudge 兜底。

### 2.4 网络探测与限速热更新

- **探测**：默认每 60s 用心跳 RPC 探活；连续 `offline_threshold`（默认 5）次失败判离线；连续 `probe_fail_threshold`（默认 5）次失败才撕控制面连接。两个阈值独立。
- **限速**：`resources.network_limit_kbps`（默认 50 KB/s）走令牌桶，逐块分段等待。服务端可通过心跳下发 `PacingPolicy` 热更新限速/节拍，`sender` 订阅 policy 变更即时生效，无需重启。

---

## 三、部署运维（一现场一套配置）

### 3.1 配置项逐项说明（`internal/config/config.go`）

| 分组 | 关键项 | 默认 | 说明 |
|------|--------|------|------|
| `mode` | — | `default` | 运行模式，见 1.1 |
| `server` | `address` | 必填 | 服务端 gRPC 地址 `<IP>:50051` |
| | `secret_key` | 必填 | HMAC 认证密钥，须与服务端一致 |
| | `path_prefix` / `tls.*` | 空/关 | 经 Nginx 前缀 / TLS 时才配 |
| `source`（default） | `host/port/user/password/database` | 必填 | 源端 MySQL |
| | `max_open_conns` / `max_idle_conns` | 2 / 1 | 压低对源库连接压力 |
| `source_minio`（default） | `endpoint/access_key/secret_key/use_ssl` | 可空 | 留空则跳过文件同步 |
| `scan`（default） | `incremental_interval_sec` | 120 | 增量扫描间隔 |
| | `batch_size` / `row_sleep_ms` | 10 / 50 | 每批行数 / 行间隔（控源库压力） |
| | `child_scan_*` / `historical_batch_size` | — | 子表/历史补扫节拍 |
| `tables`（default） | `file_block_tables` / `simple_tables` | — | 文件块主子表定义 / 普通表（可 `auto_discover`、按表覆盖 `batch_size`/`priority`/`window_hours`） |
| `sq_inbox`（sqMode） | `smartquality_dir` | OS 默认 | 只填根，子目录自动派生 |
| | `workstation_id` | — | 工作站 ID（如 `ws-现场A`） |
| | `scan_interval_sec` / `batch_size` / `max_file_size_mb` | 5 / 200 / 10 | 扫描节拍与单文件上限 |
| `data` | `block_dir` / `checkpoint_file` / `client_id[_file]` / `log_dir` / `dead_letter_dir` | `./data/*` | 状态目录；`client_id` 可固定或自动生成 |
| `resources` | `memory_limit_mb` / `network_limit_kbps` | 200 / 50 | 内存上限 / 限速 |
| | `sender_worker_count` | 1 | >1 开并发发送（内存约 +N×20MB） |
| `network` | `probe_interval_sec` / `offline_threshold` / `probe_fail_threshold` | 60 / 5 / 5 | 探测节拍与两个独立阈值 |
| `heartbeat` | `interval_sec` / `timeout_sec` | 30 / 5 | 心跳节拍与超时 |
| `retry` | `max_transient_retries` / `ack_timeout_sec` / `jitter_pct` | 10 / 120 / 25 | 瞬时重试上限 / ACK 超时 / 退避抖动 |
| `upload` | `use_chunked_for_audio` | `off` | `off/auto/on`，控制音频是否走分片续传新路径 |
| | `chunked_size_floor` / `chunk_size_bytes` | 5MB / 5MB | MinIO multipart 物理下限，不可改小 |
| `log` | `level` / `max_size_mb` / `max_files` / `console` | info/50/5/true | 日志级别与轮转 |
| `status` | `addr` | `127.0.0.1:8807` | `/api/status` UI 端点，与 SmartQuality-client 期待端口对齐 |
| `status_http_port` | — | 9091 | `/status` 运维端点，`0` 禁用 |

> 所有值支持 `${ENV_VAR:default}` 环境变量替换。**密钥、口令一律用环境变量注入，禁止把真实密钥硬编码进提交进仓库的配置。**

### 3.2 「一现场一套配置」模式

同一份二进制部署到 N 个现场，差异全部收敛在各自的 `config.yaml`。落地约定（用占位名，勿写真实现场名）：

| 差异维度 | 现场 A | 现场 N | 说明 |
|----------|--------|--------|------|
| `mode` | `default` | `sqMode` | 视现场是否有本地业务库 |
| `data.client_id` | `现场A-01` | `现场N-01` | 服务端据此隔离目标库，**全局唯一** |
| `sq_inbox.workstation_id` | `ws-现场A` | `ws-现场N` | sqMode 现场标识音频前缀 |
| `data.block_dir` 等 | 各自独立目录 | 各自独立目录 | 多实例同机时状态必须隔离 |
| 源端连接 / 共享目录 | 现场本地值 | 现场本地值 | 按现场机器填 |
| **目标库名 / MinIO bucket / 同步开关** | — | — | **不在客户端配**，由服务端 Admin 按 `client_id` 下发 |

配置模板见 `config/config.yaml`（default 全量）与 `config/config.sqmode.yaml`（sqMode 精简）。历史限速/节拍分档另有 `sync-client-限速节拍配置总表.xlsx` 供参考。

### 3.3 启动与自启

命令行只有一个参数：`-config <path>`（默认 `config/config.yaml`），另有 `-v/--version`。三平台自启（详见 `deploy/README.md`）：

| 平台 | 方式 | 自启机制 |
|------|------|----------|
| Windows | NSSM 注册为服务，或计划任务 | 开机自启；`/api/restart` 触发进程**非 0 退出（exit 2）**，由计划任务 `RestartCount` 免管理员拉起、加载新配置 |
| Linux | systemd（`Restart=always`, `RestartSec=10`） | 崩溃/退出自动重启 |
| macOS | launchd（`KeepAlive=true`, `RunAtLoad=true`） | 后台常驻 |

另有可选的**隔离块每日重试计划任务**（`deploy/windows/quarantine-daily-task.ps1` + `安装说明.txt`）：每日低峰时停 sync-client → 释放隔离块重试 → 启回 → 等 N 分钟 → 上报剩余隔离数。

### 3.4 日志与故障排查

- **日志**：`slog` 结构化输出，落 `data/logs/stdout.log`（自动轮转）与 `stderr.log`；`log.console=true` 时同时打屏。
- **状态端点**（同一 HTTP server 多路径）：`/status`（运维，含 `sender`/`uploader` 的 `transient_retry`/`ack_timeout` 计数）、`/api/status`（SmartQuality UI 用）、`/api/restart`（触发自重启）、`/debug/pprof/*`。
- **隔离块诊断工具 `unquarantine`**：与主程序同包发版，复用同一 `config.yaml` 自动定位 quarantine 目录。默认只读汇总；带 `--reason-contains`/`--block-id`/`--all` 才释放。**释放前必须先停 sync-client**，否则两进程双写状态文件导致计数器漂移。

常见现象速查：

| 现象 | 排查方向 |
|------|----------|
| 客户端不在 Admin 列表 | 网络连通、`secret_key` 是否一致 |
| `sync is disabled` | Admin 未开同步开关 |
| 心跳失败 | gRPC 端口（默认 50051）是否可达 |
| `quarantined_blocks>0` / 块卡住不发 | `unquarantine --config config.yaml` 看汇总 → 修根因 → 停服释放 |
| pending 恒为 0 但磁盘有块 | 计数器漂移，重启触发全量重扫自愈（见 2.3） |
| 音频 404 | 检查 `object_key` 是否保留 workstationID 前缀 |
| 内存/源库压力大 | 调 `memory_limit_mb` / 调高 `row_sleep_ms`、调低 `max_open_conns` |

### 3.5 已知缺陷与后续建议（弱措辞）

| 项 | 现状 | 设想方向 |
|----|------|----------|
| README 技术栈漂移 | 文档写 SQLite/Binlog，代码是文件块 + 轮询扫描 | 计划让 README 与实现对齐，避免误导接手人 |
| sqMode 上传重试队列 | 内存态，进程重启即丢，靠 SmartQuality backfill 兜底 | 设想落盘持久化重试队列 |
| pacer `SendRateEMA` | 历史一直为 0（sender 从未回调 pacer），已改由 sender 自维护 EMA 上报 | 若要激活 pacer 的 EMA 节流需单独评估，暂未启用 |
| 音频预置计数 bug | 历史上带音频的 `fb` 块曾漏计 pending（导致假"已完成"），已加磁盘核对自愈 | 保持 `.blockstore_state.json` 与磁盘一致性校验 |
| 历史扫描判定 | 曾因 simple_tables 混入导致 `IsHistoricalDone` 永远 false、realtime 卡死，已按 file_block root key 解耦 | — |
| 探测腰斩上传 | Phase C 已用控制面/数据面分离根治大部分；probe 与上传共享 conn 的遗留仍在 chunked 路径 | 设想给分片上传也独立 conn |
| 单核限制 | `GOMAXPROCS(1)` 压资源，高量现场吞吐受限 | 可调 `sender_worker_count` 并发，代价是内存 |

### 3.6 接手人从哪读起代码

建议阅读顺序：

1. `cmd/client/main.go`——**装配总线**，一眼看清所有组件如何接线、两模式如何调度（`startMode`/`OnModeChange`）。
2. `internal/config/config.go`——所有配置项、默认值、模式回退、目录派生逻辑。
3. `internal/modes/sq/`（`mode.go`→`scanner.go`→`inbox.go`）与 `internal/modes/default/`——两条采集管道各自怎么产块。
4. `internal/blockstore/store.go` + `quarantine.go`——本地块存储、去重、计数器、隔离，缓冲/续传的核心。
5. `internal/sender/sender.go`——取块、gRPC 上报、重试/隔离/死信、控制面分离。
6. `internal/heartbeat/heartbeat.go` + `proto/sync.proto`——与服务端的完整契约（状态上报字段 + 指令/策略/模式下发）。
7. `deploy/README.md`——三平台部署、自启、`unquarantine` 运维手册。

> 上手实操：本机用 `config/config.sqmode.yaml` 跑 sqMode 最快（无源端依赖），`go run ./cmd/client --config config/config.sqmode.yaml`，配合 `curl 127.0.0.1:9091/status | jq` 观察状态。

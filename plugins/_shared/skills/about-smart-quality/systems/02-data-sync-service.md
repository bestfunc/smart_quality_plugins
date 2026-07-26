# 数据同步服务（bestfunc-data-sync-server）

> 交接文档模块「数据同步服务」。本文只讲**服务端**（`sync-data/sync-server/`）。采集端与限速节拍见 [数据同步客户端](./03-data-sync-client.md)，整体拓扑见 [架构参考](../reference/02-architecture.md)，名词见 [术语表](../glossary/terms.md)。

## 一句话定义

数据同步服务是一个 **Go 编写的 gRPC 服务端**，负责把分散在各产线/门店本地 MySQL 的业务数据（含 MinIO 音频/图片文件），实时、事务级地汇聚到 SmartQuality 的**中心目标库**。它是「多源端 → 中心库」这条链路的**汇聚终点（sink）**：客户端负责"读源库、攒数据块、发出去"，服务端负责"收数据块、校验去重、写目标库、回填文件 URL"。

- **块（block）**：同步的最小传输/事务单位。一个块 = 一张根表的若干行 + 关联子表若干行 + 关联文件引用，打包成一段 JSON 元数据（payload）加可选文件字节流。服务端以"整块"为事务边界写入目标库，保证一致性。
- **源端 / 目标库**：源端指某产线本地的业务 MySQL（如 SmartQuality 本地库）；目标库指中心侧的汇聚 MySQL，按客户端隔离到不同 database。

> ⚠️ **给接手人的第一提醒**：仓库根目录的 `README.md`、`architecture-docs/`、`docker-compose.yaml` 描述的是一版**更宏大的设想架构**（Redis Stream 缓冲、16 分片聚合、64 Slot 路由、Prometheus、独立 FileSync 服务）。**当前实际运行的 `sync-server` 二进制并未实现这些**——真实代码是一条更简单直接的"gRPC 收块 → 校验去重 → 写 MySQL/MinIO"链路。本文只描述**代码能证实**的实现，并在末尾专列文档漂移清单。

---

## 一、服务架构与数据流

### 1.1 真实数据流（以代码为准）

```
源端本地 MySQL (Binlog / 全量扫描)
   │  客户端采集、攒块、限速
   ▼  gRPC 双向流 / 一元 RPC（HTTP/2 h2c 明文）
┌──────────────────────── sync-server ────────────────────────┐
│ Receiver(BlockReceiver)  收流式块 → 落临时目录 data/temp/    │
│      │                                                        │
│ Validator  block_id 幂等去重 + SHA256 校验 + 文件校验         │
│      │                                                        │
│ Writer(BlockWriter)  阶段1 传文件到 MinIO → 阶段2 建库建表    │
│      │               → 阶段3 事务 UPSERT + 回填文件 URL       │
│      ▼                                                        │
│ 目标 MySQL 8.0        目标 MinIO（对象存储）                   │
│                                                              │
│ 旁路: ClientStore(在线状态) / SyncState(每客户端配置,JSON)   │
│       CheckpointSampler(采样目标库 MAX(updateTime))          │
│       Admin HTTP(:8090, React 后台 + REST)                   │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 核心模块（`sync-server/internal/`）

| 模块 | 文件 | 职责 |
|------|------|------|
| **receiver** | `receiver/block_receiver.go` | gRPC 全部 RPC 的实现（`SyncBlocks`/`Heartbeat`/分片续传/`CommitBlock`）；收流、落临时文件、串起校验→写入 |
| **validator** | `validator/validator.go` | block_id 幂等去重（内存 map + JSON 落盘）、块 SHA256 校验、文件校验 |
| **writer** | `writer/block_writer.go` | 两/三阶段写目标库：MinIO 上传 → `CREATE IF NOT EXISTS` 建表 → 事务 UPSERT → 回填 URL；瞬时错误分类 |
| **writer(multipart)** | `writer/multipart_writer.go` | 音频大文件走 MinIO multipart 分片合并 |
| **uploadstate** | `uploadstate/state.go` | `sync_meta.sync_upload_state` 表 CRUD，记录分片上传进度（唯一落 MySQL 的元数据表） |
| **syncstate** | `syncstate/state.go` | 每客户端配置（目标库名、bucket、开关、节拍策略、mode）持久化到 `sync_state.json` |
| **clientstore** | `clientstore/store.go` | 内存态在线状态（心跳驱动，90 秒超时判离线） |
| **admin** | `admin/server.go` | 管理后台 HTTP：静态前端 + REST API + checkpoint 采样器 |
| **admin(sampler)** | `admin/checkpoint_sampler.go` | 定期查各客户端目标库 `MAX/MIN(updateTime)`，算"当前断点"和"追赶倍率" |
| **filesync** | `filesync/*.go` | URL 重写/映射的独立实现——**当前未接入 main.go**（遗留/预研，实际重写在 writer 内） |
| **config** | `config/config.go` | YAML 加载 + `${ENV:default}` 变量替换 + 默认值 |

### 1.3 技术栈

| 分类 | 选型 | 备注 |
|------|------|------|
| 语言 | Go 1.24（Dockerfile）/ 1.21+（README） | 单静态二进制 |
| 通信 | gRPC over HTTP/2 **h2c**（明文，非 TLS） | `golang.org/x/net/http2/h2c`；便于经 Nginx 按 `/grpc/` 前缀分流 |
| 目标数据库 | MySQL 8.0 | 代码只注册 `go-sql-driver/mysql`；PostgreSQL 仅见于 `docs/postgresql-support-migration-plan.html`，**设想中，未实现** |
| 对象存储 | MinIO（`minio-go/v7`，S3 兼容） | 存音频/图片等关联文件 |
| 元数据 | 本地 JSON 文件为主 + `sync_meta.sync_upload_state` 一张表 | 见 §五 |
| 前端 | React 18 + Umi Max + Ant Design | 构建产物用 `go:embed` 打进二进制 |
| 序列化 | protobuf（gRPC 帧）+ JSON（块 payload） | 块内容是 JSON，不是 protobuf |

---

## 二、同步协议与数据格式（gRPC）

服务只暴露一个 gRPC service `syncpb.SyncService`（定义见 `sync-server/proto/sync.proto`），共 6 个 RPC：

| RPC | 方向 | 用途 |
|-----|------|------|
| `SyncBlocks` | 客户端流式 → 服务端 | **主同步路径**：一个块的元数据 + 文件字节分块（32KB/帧）流式发来，服务端 `SendAndClose` 回一个 ACK |
| `Heartbeat` | 一元 | 保活 + 客户端状态上报 + 服务端下发指令（暂停/恢复、节拍策略、模式、能力协商） |
| `GetUploadProgress` | 一元 | 音频分片续传：客户端重连后问"这个文件传到第几片了" |
| `SendChunk` | 一元 | 上传一个分片（4MB/片），服务端转发到 MinIO multipart |
| `CompleteUpload` | 一元 | 通知所有分片传完，触发 MinIO `CompleteMultipartUpload` |
| `CommitBlock` | 一元 | **v3.4 新路径**：文件先经分片续传落 MinIO，再用一次原子提交写 MySQL 行 + 引用对象，`block_id` 强幂等 |

### 数据格式：块 payload（JSON）

`SyncBlockRequest.payload` 是一段 JSON，服务端 `writer.Block` 结构解析（示例）：

```json
{
  "block_id": "blk_...",
  "block_type": "file_block",
  "mode": "sqMode",
  "root_table":    { "database": "...", "table": "detect", "create_sql": "CREATE TABLE ...", "rows": [ {...} ] },
  "related_tables":[ { "database": "...", "table": "detect_item", "fk_field": "detectId", "rows": [ {...} ] } ],
  "files":         [ { "ref_table": "detect", "ref_row_id": 123, "field": "audioUrl",
                       "source_url": "http://src/bucket/key", "bucket": "...", "file_checksum": "sha256:..." } ]
}
```

- `checksum` 字段（帧上带）= 对 `block_data` 结构（不含 files 字节）做 `sha256:<hex>`；服务端重算比对，不符即 `CHECKSUM_MISMATCH` 拒绝。
- `create_sql`：源表的 `SHOW CREATE TABLE`，服务端据此在目标库自动建表。
- **错误码**（`SyncBlockResponse.error_code`）：`CHECKSUM_MISMATCH` / `SCHEMA_MISMATCH` / `DATA_INVALID` / `FK_VIOLATION` / `ALREADY_PROCESSED` / `INTERNAL` / `SYNC_DISABLED`；瞬时错误改用 gRPC `Unavailable` 状态码让客户端重试。

### 鉴权

gRPC 拦截器（`receiver/auth_interceptor.go`）在 metadata 里校验客户端 token。两种模式：配置 `server.secret_key` 时用 HMAC 派生（`NewTokenStoreWithSecret`），或配 `server.tokens` 白名单逐个匹配。校验通过后把 `client_id` 注入 context，后续所有写入以此为准（防客户端伪造 `client_id` 越权写别人的库）。

---

## 三、四种同步来源（服务端视角统一为"块"）

客户端区分四种数据来源，但**到了服务端它们都是块**，走同一套 `SyncBlocks`/`CommitBlock` 收敛：

| 来源 | 术语解释 | 服务端如何识别 |
|------|----------|----------------|
| **增量（Binlog）** | 客户端读源库 MySQL Binlog（二进制变更日志）实时捕获 INSERT/UPDATE | 无特殊标记，普通块，UPSERT 幂等写入 |
| **全量** | 客户端全表扫描历史存量数据补齐 | 同上；块可能行数多，写入超时按行数追加预算 |
| **文件（MinIO）** | 行里引用的音频/图片，随块附带字节流或经分片续传 | 块 `files[]` 数组；大文件走 `SendChunk`+`CommitBlock` |
| **离线（NDJSON）** | 断网期客户端把变更存成 NDJSON 文件，恢复后回放 | 服务端无感——仍是普通块 |

> 换言之，服务端不关心块"从哪来"，只关心块的 `mode`（`default` / `sqMode`）以决定写哪个目标库/bucket。**离线 NDJSON 的攒/回放逻辑全在客户端**，本服务不含离线导入模块。

---

## 四、断点续传与冲突处理

### 4.1 断点续传（音频分片）

大音频文件用 **MinIO multipart（分片上传）**+ `sync_meta.sync_upload_state` 表实现断点续传：

1. 客户端算出 `file_key`（源端文件唯一键），调 `GetUploadProgress` 问服务端进度。
2. 服务端查 `sync_upload_state`：无记录→从 part 1 开始并初始化 MinIO multipart；有记录→返回 `acked_part_count`，客户端从 `acked+1` 续传。
3. 每片经 `SendChunk` 转发 MinIO，服务端按 part_number **幂等累积 ETag**（重发同片 ETag 相同，覆盖无害）。
4. `CompleteUpload` 按 part 升序合并；`upload_id` 过期则回 `expired=true`，客户端从头再来。

`sync_upload_state` 关键列：`(client_id, file_key)` 主键、`upload_id`、`acked_part_count`、`expire_at`（默认 +7 天）、`status`（`inflight`/`done`/`completed_by_commit`/`expired`/`aborted`）。孤儿 GC（`upload.orphan_gc`，默认 24h TTL、60min 扫描）设想清理提交失败留下的 inflight 对象——**注意：GC 后台任务在 config 里有配置项，但 main.go 未见启动该 goroutine，实际清理能力待接手人核实**。

### 4.2 块级断点

- **块内幂等**：`block_id` 去重（§五）保证同一块重传不会写两次。
- **写失败回退**：`WriteBlock` 失败时 `validator.Unmark(block_id)` 撤销去重标记，客户端下次重传可重走完整流程。
- **优雅停机的取舍**：停机用 `grpcServer.Stop()`（暴力断连，非 `GracefulStop`——h2c 模式下后者会 panic），正在传的块被切断，**靠客户端重传兜底**。

### 4.3 冲突处理

| 冲突类型 | 处理策略 |
|----------|----------|
| 主键重复（事实表） | `INSERT ... ON DUPLICATE KEY UPDATE` 全字段覆盖（远端为权威） |
| 主键重复（字典表） | 块标 `if_absent=true` → 改用 `INSERT IGNORE`，撞主键静默跳过，不覆盖人工维护数据（product/detect_station 等） |
| 目标表缺列 | "同步表结构"按钮触发 `ReconcileSchema`：解析源 DDL，**只加缺失列**（`ALTER ADD COLUMN`），绝不改类型/删列/动索引 |
| 字符集/排序规则冲突 | 建表时把源 DDL 的 `COLLATE utf8mb4_*` 统一改写为 `utf8mb4_general_ci`，`utf8` 升 `utf8mb4`，避免跨库 JOIN 报 1267 |
| 目标库名冲突（多客户端混写） | `default` 模式**强制**把块内 database 覆盖为 admin 配的 `target_database`，隔离各客户端；`sqMode` 保留 per-row 路由 |
| MySQL 暂停/MinIO 抖动 | 归类为**瞬时错误**（`ErrTransient`：ctx 超时 / connection refused / 1205 锁等待 / 2006 gone away 等），回 gRPC `Unavailable`，客户端重试而非进隔离区 |

---

## 五、三层去重

服务端从三个粒度保证"重复不落库"：

| 层 | 位置 | 机制 |
|----|------|------|
| **块级** | `validator.Validate` | `block_id → 处理时间` 内存 map；命中返回 `ALREADY_PROCESSED`。map 每分钟落盘 `processed_blocks.json`，重启恢复；按 TTL（默认 24h）清理过期项 |
| **文件级** | `writer.uploadFile` | 上传前 `StatObject` 查目标对象，若已存在且**大小一致**则跳过 PUT，直接复用 URL |
| **行级** | `writer.upsertRow` | `UPSERT`/`INSERT IGNORE` 天然幂等，同一行重复写结果不变 |

分片续传另有一层 **part 级去重**：`SendChunk` 按 part_number 幂等累积 ETag，重发同片不产生重复分片。

---

## 六、URL 透明重写

源端行里的文件 URL 指向**源端 MinIO**（如 `http://源地址/bucket/key`），汇聚后必须改指向**中心 MinIO**，否则中心侧读不到文件。服务端在写入时透明完成：

1. `uploadFile` 从 `source_url` 提取 object key（`extractObjectKey`，保留目录结构），把文件 PUT 到**目标 bucket**。
2. 生成新 URL：`/<目标bucket>/<object_key>`。
3. 事务内执行 `UPDATE <表> SET <field>=<新URL>, bucket=<目标bucket> WHERE id=<ref_row_id>`，把行里的 URL 字段就地改写。

对上层业务系统完全透明——行数据同步过来时，URL 已指向中心存储。`CommitBlock` 路径同理（`WriteBlockWithRefs`），只是文件已在 MinIO，改为 `StatObject` 校验存在 + size，再回填 URL。

---

## 七、任务调度

服务端没有独立的"任务队列/调度器"，调度通过**心跳下发**和**采样循环**两条旁路完成：

- **心跳指令下发**（`Heartbeat`）：服务端比对客户端上报的版本号，按需下发——
  - `command`：`pause` / `resume`（一次性指令，`ConsumePendingCommand` 消费即清）；`rescan_file_block:<时间>`（回退游标重扫）。
  - `pacing_policy` + `pacing_version`：完整节拍/限速/资源策略快照，版本不一致才下发。
  - `mode` + `sq_rate_tier` + `mode_version`：切换 default/sqMode 及限速档位（1-9 分档，10 不限速）。
  - `server_capabilities`：v3.4 能力协商，仅对灰度名单（`upload.gray_clients`）客户端广播 `commit_block_v1`，命中才切 chunked 直传，否则自动回退 `SyncBlocks`。
- **CheckpointSampler**（`admin/checkpoint_sampler.go`）：后台 goroutine 每 `checkpoint_sample_interval_sec`（默认 60s）对每个启用的客户端目标库查 `MAX/MIN(updateTime)`（表/字段默认 `detect`/`updateTime`），持久化到 `checkpoint_samples.json`，供后台展示"当前断点""追赶倍率"，重启即可恢复。
- **后台常驻循环**：`ClientStore.MonitorLoop`（判离线）、`Validator.CleanupLoop`（清去重表 + 落盘）。

---

## 八、客户端状态与配置存储

服务端元数据**以本地 JSON 文件为主，仅一张 MySQL 表**（不用 Redis）：

| 存储 | 内容 |
|------|------|
| `sync_state.json`（`syncstate`） | 每客户端配置：`enabled` 开关、`target_database`、`minio_bucket`、`pacing`（节拍策略 + 版本）、`mode`/`sq_rate_tier`/`sq_target_database`/`sq_minio_bucket`、累计字节/文件数 |
| `processed_blocks.json`（`validator`） | 去重表：`block_id → 处理时间` |
| `checkpoint_samples.json`（`sampler`） | 各客户端断点样本 + 最后心跳缓存 |
| 内存（`clientstore`） | 在线状态、心跳快照、scanner/隔离/优先窗口进度（重启丢失，靠心跳/采样兜底） |
| `sync_meta.sync_upload_state`（MySQL） | 分片上传进度（唯一持久化到 DB 的元数据） |

> 开关约束：`SetClientEnabled(true)` 要求该客户端已配好 `target_database` 和 `minio_bucket`，否则拒绝——防止启用后块无处可写。`SyncBlocks` 开头也会查 `IsClientEnabled`，关闭时直接回 `SYNC_DISABLED`。

---

## 九、管理后台与 API

### 后台（服务端 React）

- 技术：React 18 + Umi Max + Ant Design，构建产物 `go:embed` 进二进制，由 admin HTTP（`:8090`）托管，SPA history fallback。
- 页面：源码 `web/src/pages/` 实际只有 **ClientList（客户端列表）** 和 **ClientEdit（客户端配置编辑）** 两个页面（README 称"10 页面"为设想值）。
- 预览期后台**未强制登录**（`basicAuth` 已实现但注释掉），仅内网访问；CORS 放开任意来源。

### REST API（`admin/server.go`，标准库 `net/http`，非 Gin）

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/api/v1/health` | 健康检查（版本/commit） |
| GET | `/status` | writer 瞬时错误计数（`db_exec_timeout` / `minio_part_timeout`） |
| GET | `/api/v1/clients` | 在线状态列表 |
| GET | `/api/v1/clients/detail` | 合并后的客户端详情（在线/开关/断点/追赶倍率/sqMode 指标） |
| GET/POST/DELETE | `/api/v1/client-config/{id}` | 读/写/删客户端配置（写会校验优先窗口、自增 pacing_version） |
| POST | `/api/v1/client-config/{id}/enabled` | 开关同步 |
| POST | `/api/v1/client-config/{id}/mode` | 切 default/sqMode |
| POST | `/api/v1/client-config/{id}/sq-rate-tier` | 设 sqMode 限速档位 |
| POST | `/api/v1/client-config/{id}/sq-target` | 设 sqMode 目标库/bucket |
| POST | `/api/v1/client-config/{id}/sync-schema` | 对该客户端目标库做 additive 列对账 |
| POST | `/api/v1/clients/rescan` | 排一条 `rescan_file_block` 指令（body 带 `since` RFC3339） |
| POST | `/api/v1/flush-schema-cache` | 清建表缓存（运维 DROP 表后强制重建） |
| GET | `/api/v1/quarantine/{id}` | 隔离块摘要（来自心跳上报） |
| GET | `/api/v1/sq/trace` | 单记录追踪——**当前是 stub，返回占位** |
| POST | `/api/v1/sq-admin/reset-test` | 测试用：清空 sqMode 目标库 + bucket + upload_state（**仅测试环境**） |

---

## 十、部署

镜像三阶段构建（`docker/docker-image-server/Dockerfile`）：Node 构前端 → Go 编译静态二进制 → Alpine 运行，时区固定 `Asia/Shanghai`。构建脚本 `build.sh` 用 `docker save` 导出 tar，拷到目标机 `docker load` 后 `docker run`。`deployments/docker-compose.yaml` 另含一套本地依赖栈（含 MySQL/MinIO/Redis）。

### 端口

| 端口 | 服务 | 状态 |
|------|------|------|
| 50051 | gRPC 数据同步（h2c） | ✅ 实际监听 |
| 8090 | 管理后台 HTTP + REST | ✅ 实际监听 |
| 9090 | Prometheus metrics | ⚠️ Dockerfile/compose 声明，**二进制未实现** |
| 8081 | 独立 FileSync HTTP | ⚠️ 同上，未接入 |

### 关键配置项（`config/config.yaml`，`${ENV:default}` 支持环境变量覆盖）

| 项 | 默认 | 说明 |
|----|------|------|
| `server.grpc_port` | 50051 | gRPC 端口 |
| `server.max_connections` | 50 | gRPC 最大并发流 |
| `server.secret_key` | — | HMAC token 派生密钥（配置文件含默认值，**生产必须改**） |
| `target.host/port` | 127.0.0.1:3306 | 目标 MySQL |
| `target_minio.endpoint` | 127.0.0.1:9000 | 目标 MinIO |
| `writer.exec_timeout_sec` | 90 | 单条 SQL 超时 |
| `writer.exec_timeout_per_1k_rows` | 5 | 每千行追加秒数 |
| `writer.minio_part_timeout_sec` | 120 | MinIO 单 part 超时 |
| `upload.enable_chunk_protocol` | true | chunked 协议总开关 |
| `upload.gray_clients` | `[]` | 灰度名单（空=全员走旧 `SyncBlocks`） |
| `admin.checkpoint_sample_interval_sec` | 60 | 断点采样间隔 |
| `data.processed_blocks_ttl` | 24h | 去重表保留时长 |

- gRPC 帧上限调到 16MB（默认 4MB 会因分片 8MiB 报 `ResourceExhausted`）。
- 支持 Nginx 按 `/grpc/` 前缀分流：服务端会自动 strip 前缀。

---

## 十一、性能与容量现状

以代码/文档能证实的口径：

- **并发**：`max_connections` 默认 50 条并发 gRPC 流，一客户端一条流；目标 MySQL 连接池默认 `max_open=10 / max_idle=5`。
- **写入模型**：单块单事务串行 UPSERT，无批量攒写、无并发分片写目标库（README 的"16 分片聚合/64 Slot"未实现）。吞吐主要受客户端限速（节拍策略）与目标 MySQL 单事务速度约束。
- **瞬时错误可观测**：`/status` 暴露 `db_exec_timeout` / `minio_part_timeout` 累计计数，用于判断目标库/MinIO 是否在抖。
- **速率报告**：`docs/` 下有 `zass-01`/`zx999-01`/`多客户端同步速率分析报告.html` 等实测材料，可作为容量基线参考（数据随现场而变，接手后以最新报告为准）。

---

## 十二、监控（可观测现状）

> README/compose 写的 "Prometheus 32+ 指标" 是**设想**。当前二进制**未集成 Prometheus**，也未监听 9090。真实可观测手段是：

| 手段 | 内容 |
|------|------|
| `GET /status` | writer 瞬时错误计数（DB/MinIO） |
| `GET /api/v1/health` | 版本、commit、存活 |
| `GET /api/v1/clients/detail` | 每客户端在线/断点/追赶倍率/隔离块/sqMode 进度 |
| 结构化日志 | 标准库 `slog`，块级 Info/Warn/Error，含 `block_id`/`client_id`；中途断流、瞬时错误、隔离原因都有 trail |

Prometheus 接入可作为**后续可探索方向**（compose 已留端口位）。

---

## 十三、已知缺陷 / 未完成 / 文档漂移

1. **文档-实现严重漂移（最高优先）**：`README.md`、`architecture-docs/01-09`、`docker-compose.yaml` 描述的 Redis Stream 缓冲、多分片聚合、64 Slot 路由、Prometheus、独立 FileSync 服务，**均未在当前 `sync-server` 二进制实现**。compose 甚至 `depends_on` 一个跑起来却不被连接的 Redis。接手时**以 `sync-server/internal/*.go` 为唯一事实来源**。
2. **`filesync` 包未接线**：`internal/filesync/`（rewriter/receiver/url_mapping）未被 main.go 引用，URL 重写实际在 `writer` 内完成。该包是遗留/预研代码。
3. **孤儿 GC 未确认启动**：`upload.orphan_gc` 有配置项，但 main.go 未见对应后台 goroutine，提交失败的 MinIO inflight 对象清理能力待核实。
4. **PostgreSQL 目标库仅停留在计划**：代码只连 MySQL；`docs/postgresql-support-migration-plan.html` 是迁移设想。
5. **stub 接口**：`/api/v1/sq/trace` 单记录追踪返回占位，真实查询逻辑未落地。
6. **后台无登录**：预览期 admin `basicAuth` 被注释、CORS 全开，仅靠内网隔离，生产化前需补回。
7. **停机切断在途块**：`Stop()` 硬断连，完全依赖客户端重传，无服务端侧持久化排队。
8. **端口声明冗余**：Dockerfile `EXPOSE 9090 8081` 与实际监听不符，易误导运维。

### 后续可探索方向（弱措辞）

- 计划中：接入 Prometheus/告警，补齐孤儿 GC，落地 `sq/trace` 排障。
- 设想中：目标库多路并发写入以提吞吐；后台补登录鉴权与审计。
- 在调研：文档与实现对账，删除/归档遗留 `filesync` 与失效架构文档，降低交接认知成本。

---

## 十四、接手人从哪读起代码

按这个顺序，一天内能建立完整心智模型：

1. **`cmd/server/main.go`（231 行）**——总装线：看清启动了哪些 goroutine、各模块怎么注入。**这是唯一权威的"实际架构图"**。
2. **`proto/sync.proto`**——协议契约：6 个 RPC 和块 JSON 结构，读完就懂客户端-服务端怎么对话。
3. **`internal/receiver/block_receiver.go`**——请求入口：`SyncBlocks`（主路径）、`Heartbeat`（调度下发）、`CommitBlock`（v3.4 新路径）三个方法串起全链路。
4. **`internal/writer/block_writer.go`**——落库核心：三阶段写入、UPSERT、建表、URL 重写、瞬时错误分类都在这。
5. **`internal/validator/validator.go`**——去重与校验，短小。
6. **`internal/syncstate/state.go` + `internal/config/config.go`**——配置与状态模型，理解"每客户端怎么隔离"。
7. **`internal/admin/server.go`**——后台 API，对照 §九 表逐个看 handler。
8. 最后再回头**对照读** `README.md` / `architecture-docs/`，**带着"这是设想稿"的心态**辨别哪些是真的、哪些是 aspirational。

> 快速验证链路：本地起 MySQL + MinIO，`go run ./cmd/server`，在源库 INSERT 一行，看目标库是否出现同名 database 下的对应行、URL 是否已重写为中心 MinIO 路径。

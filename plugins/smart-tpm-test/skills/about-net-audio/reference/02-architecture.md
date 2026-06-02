# 系统架构

## 一句话定义

net_audio 是 **"嵌入式盒子里跑两个 Go 进程 + 一份 Vue3 前端"** 的双进程架构。**smartaudio** 负责音频采集、缓冲与对外算法接口；**smartaudio-admin** 负责 Web 管理与硬件参数调校。

## 双进程拓扑

| 进程 | 端口 | systemd unit | 配置文件 | 仓库内位置 |
|---|---|---|---|---|
| `smartaudio`（主程序） | 8090 | `SmartAudio.service` | `config.yml` | 仓库根（Go module `smartaudio`） |
| `smartaudio-admin`（Web 后台） | 8091 | `SmartAudioAdmin.service` | `admin-config.yml` | `admin/`（Go module `smartaudio-admin`） |

部署路径固定 `/home/rock/audio/`，两个进程都以 root 启动。**没有反向代理 / Nginx 中间层** —— 客户端直连 8090（算法/stream）或 8091（Web 管理）。

为什么拆两个进程：
- 主程序生命周期跟"采集"绑定，重启 = 录音中断
- 管理后台需要频繁更新（Web UI 升级、API 加字段），不希望牵动采集
- admin 端必须有动配置 / 重启主程序的权限（双进程时它就是 systemctl 调用方而非"自己重启自己"，逻辑干净）

## 数据流（音频从麦克风到上游算法的全链路）

```
[Rock Pi S 主板 I2S 接口]
   ↓ (硬件)
[4 通道麦克风阵列板] —— 物理标签 A B C D
   ↓
[ALSA: dummy-card hw:1,0]
   ↓
[PortAudio 输入回调] ── internal/audio/recorder.go::StartRecording()
   ↓ 每 10ms 一帧（441 sample/ch @ 44.1kHz）
[BufferPool] ── internal/audio/buffer_pool.go
   ├──→ 实时分发到 WebSocket 订阅者 (stream/manager)
   ├──→ 写入 RingBuffer (storage/ringbuffer.go → recordings.narb)
   └──→ [算法按需切片] ←── /api/v1/stop_and_fetch
        ↓
        [按 detectStageID 时间窗 → PCM 数组 → WAV 字节]
        ↓
        [HTTP 响应：单通道 audio/wav，多通道 multipart/form-data]
        ↓
        上游算法 / SmartPLC

[另一条独立路径：legacy 录音队列]
[/api/v1/start_record + /api/v1/upload_audio]
   ↓ 入 cacheFileStream
   ↓
[upload.intervalUpload ticker] —— internal/upload/file_uploader.go
   ↓ Upload.Enable=true 时
[POST 到 Upload.Endpoint] —— 配置里的 web 接收方
```

**两条音频路径互不干扰**：算法直调走 `stop_and_fetch` → BufferPool → 即时返回；老的 record → upload 链路走 cacheFileStream → 后台 ticker。详见 [reference/04-key-concepts.md §音频生命周期](./04-key-concepts.md)。

## 主程序 internal/ 模块清单

源头：`docs/PROJECT_OVERVIEW.md`，已根据 v3.5 实际代码核对。

| 模块 | 主要文件 | 职责 |
|---|---|---|
| `app/` | `application.go` | 应用启动、CLI 子命令注册、各子系统初始化 |
| `audio/` | `recorder.go` / `stream_buffer.go` / `buffer_pool.go` | PortAudio 采集、缓冲池、流式分发 |
| `controller/` | `controller.go` / `cache_file.go` | 多声道控制器，将音频按声道拆分、维护 cacheFileStream |
| `stream/` | `manager.go` | 实时音频流 WebSocket 会话管理 |
| `storage/` | `ringbuffer.go` | 文件级环形缓冲（recordings.narb，覆盖式写入） |
| `upload/` | `file_uploader.go` / `db_report.go` | 异步上传队列（legacy 路径） |
| `api/` | `server.go` | 本地 HTTP / WebSocket API |
| `conf/` | `index.go` / `common/interface.go` / `local/` / `remote/` | YAML 配置加载 + 远程动态下发 |
| `discovery/` | `discovery.go` | mDNS 广播（zeroconf.RegisterProxy，每台用唯一 SN host） |
| `device/` | `device.go` | 输入设备枚举 + SN 读取 |
| `timesync/` | `time_sync.go` / `time_adjuster_*.go` | 客户端↔服务器时间同步 |
| `clock/` | `clock.go` | 时钟抽象 |
| `channel/` | `label.go` | **A/B/C/D 物理标签 ↔ 内部声道号权威映射** |
| `register/` | `register.go` | 设备向后端注册（上报 SN / 版本） |
| `persistence/` | `persistence.go` | 本地持久化运行数据（.smartaudio） |
| `script/` | `script.go` | 执行内置脚本（如硬件初始化） |
| `writer/` | `file_writer.go` / `tcp_writer.go` / `udp_writer.go` / `chunk_writer.go` | 多种音频输出后端 |

主程序 CLI 子命令（见 `application.go`）：

| 子命令 | 说明 |
|---|---|
| `devices` | 列出输入设备（拿 `device_index`） |
| `run` | 前台运行采集 |
| `install` / `start` / `stop` / `status` / `uninstall` | 系统服务生命周期 |
| `version` | 输出 Version / Build / Commit / SN |

## admin 进程模块清单

源头：`admin/internal/`。

| 模块 | 职责 |
|---|---|
| `app/app.go` | 启动入口、版本内嵌（`go:embed version.txt`） |
| `api/router.go` | 路由表定义（JWT 鉴权层 + 无鉴权层） |
| `api/handlers/` | 各 endpoint handler（auth / service / config / logs / system / storage / debug / sensitivity / network） |
| `api/static/` | embed 的 Vue3 前端 dist 产物（`go:embed all:static`） |
| `service/smartaudio.go` | 管理主进程：start/stop/restart/status |
| `service/config_manager.go` | yaml 读写 + 备份 |
| `service/sensitivity.go` | 灵敏度更新（v3.5.3 路径：写 yaml + exec restart 脚本） |
| `service/network.go` | netplan 静态 IP 配置管理 |
| `service/log_manager.go` | 日志查询 / 下载 / 清理 |
| `service/system_monitor.go` | CPU/内存/磁盘 |
| `service/storage_reader.go` | 浏览 RingBuffer / 下载历史音频片段 |

## 关键 API 路由摘要

完整对接文档：[reference/11-ai-integration.md](./11-ai-integration.md)。

**主程序 8090（无 / 部分鉴权，详见各 endpoint 注释）**：

| 路径 | 协议 | 用途 |
|---|---|---|
| `/api/v1/time_sync` | HTTP POST | 客户端 ↔ 盒子时钟同步（强前置） |
| `/api/v1/stop_and_fetch` | HTTP POST | **算法直调音频字节流（核心 API）** |
| `/api/v1/start_record` / `/upload_audio` / `/stop_record` | HTTP POST | Legacy 录音队列接口 |
| `/api/v1/stream/live` | WebSocket | 实时音频流（订阅指定通道，PCM/WAV chunk） |
| `/api/v1/stream/sessions` / `/session/:id` | HTTP | 流会话查询 / 关闭 |

**Admin 8091**：

| 路径 | 鉴权 | 用途 |
|---|---|---|
| `/api/auth/*` | 无 / 登录态 | 登录 / 登出 / 刷新 |
| `/api/service/*` | 登录 | start/stop/restart/status/version |
| `/api/config` | 登录（PUT 管理员） | yaml 全文 读 / 写 / 备份 |
| `/api/audio/sensitivities` | **无** | 灵敏度 读 / 写（A/B/C/D 双轨入参） |
| `/api/network` | **无** | 静态 IP 读 / 写（netplan） |
| `/api/logs/*` / `/api/system/*` / `/api/storage/*` | 登录 | 日志、系统监控、存储浏览 |
| `/api/debug/stream` | 登录 | WebSocket 调试音频流（前端调试用） |
| `/api/debug/sensitivities` | 登录 | 灵敏度兼容路径（同 /api/audio） |

`/api/audio/sensitivities` 和 `/api/network` **无鉴权**是 v3.5.3 产品决策，理由：现场算法/PLC 直接调用便利性优先；前提是这套设备只暴露在产线内网。

## mDNS 发现

每台盒子启动时通过 `zeroconf.RegisterProxy` 广播：

```
服务类型：_smartaudio._tcp.local.
实例名：  SmartAudio-<SN>
主机名：  smartaudio-<SN>.local.   ← 用 SN 派生，确保多盒子唯一
端口：    8090 (配置 discovery.api_port)
TXT：     sn=<SN> model=<model> fw=<version>
```

**只广播主程序 8090**，不广播 admin 8091（约定：admin 端口 = api_port + 1）。

Why 用 `RegisterProxy` 而不是普通 `Register`：出厂镜像 hostname 都克隆为 `rockpis`，普通 Register 会用系统 hostname → 多盒子撞库。RegisterProxy 允许指定不同的 mDNS host 名而不改系统 hostname。详细背景见 [reference/10-deployment-modes.md](./10-deployment-modes.md)。

## 配置文件结构

主程序 `/home/rock/audio/config.yml`，几个核心 section：

```yaml
audio:                # 采样率、通道数、splits、灵敏度
sensitivities: [..]   # 4 通道的 1-255 灵敏度值
buffer_pool:          # BufferPool 时长 / 通道
storage:              # RingBuffer 启用 / 文件路径 / max_size
upload:               # legacy 上传路径配置（enable / endpoint / interval）
discovery:            # mDNS 广播开关 + api_port
realtime:             # 上游 stream 接收方地址
remote_config:        # 远程配置拉取（applyInfo 等）
report:               # 报告上报
logger:               # 日志级别 / 文件路径
persistence:          # 持久化运行数据
system:               # 灵敏度脚本路径等
```

admin 的 `/home/rock/audio/admin-config.yml` 较薄：监听端口、JWT secret、登录账号。

## 现场已知约束

> 详见 [reference/10-deployment-modes.md](./10-deployment-modes.md)，这里只列纲。

- 出厂镜像 hostname / 静态 IP / machine-id / SN 都克隆，**只有 MAC 是唯一标识**
- Rock Pi S **无 RTC**，断电会丢系统时间
- 麦克风采集硬件存在**个体差异**（部分板 Input overflowed 频率高，软件不可修复）

# 术语与缩写表

按字母 / 拼音排序。**首次在其它文件出现的专有名词**应该收录在这里。

---

## A

**A/B/C/D 物理标签**
盒子正面从左到右的 4 个物理麦克风位置。**与内部声道号不顺序对齐**：A=ch3 / B=ch4 / C=ch1 / D=ch2。权威翻译代码：`internal/channel/label.go`。

**ALSA** (Advanced Linux Sound Architecture)
Linux 音频驱动框架。net_audio 通过 PortAudio 间接调用 ALSA 访问 rk3308 主板上的 I2S 麦克风设备。

**api_port**
`discovery.api_port` 配置字段。mDNS 广播给客户端的 HTTP API 端口号，必须跟实际监听端口一致（默认 8090）。

**audio.sensitivities**
`config.yml` 的 4 通道灵敏度数组，每元素 ∈ [1, 255]。按内部声道顺序 `[ch1, ch2, ch3, ch4]`。

**Avahi**
Linux 下的 mDNS 实现（Bonjour 等价）。客户端用 `avahi-browse` 发现 SmartAudio 盒子。

---

## B

**BufferPool**
[reference/04-key-concepts.md §2](../reference/04-key-concepts.md)。内存里的最近 N 秒音频环形缓冲，按 channel 切片。所有"按时间窗取音频"的接口（如 stop_and_fetch）从这里取数据。位置：`internal/audio/buffer_pool.go`。

**buffer_pool.duration**
BufferPool 的保留时长，单位秒（默认 30）。请求超出此时长会返 `STAGE_NOT_FOUND`。

---

## C

**cacheFileStream**
[reference/04-key-concepts.md §4](../reference/04-key-concepts.md)。legacy 录音队列，按 channel 切分的 CacheFile 数组。`/api/v1/start_record` + `/upload_audio` 入队，后台 ticker (`Upload.Enable=true` 时) 扫 state=Wait 项 POST 到 `upload.endpoint`。位置：`internal/controller/cache_file.go`。

**channel** （内部声道号）
1 / 2 / 3 / 4，跟代码层走的索引。**注意**不等于物理位置（详见 A/B/C/D）。

**channel_labels**
接口入参字段，按物理位置传 `["A", "C"]` 等。内部翻译成 channels。新接入推荐用这个，避免搞错映射。

**CGO**
Go 调用 C 代码的桥接。net_audio 主程序需要 CGO（PortAudio 是 C 库），admin 是纯 Go 不需要 CGO。

---

## D

**dBA**
A 加权分贝。admin 调试 UI 里显示的响度值。

**detect** / **detectStageID**
stop_and_fetch 请求体的字段。`detect.detectStageID` 是算法侧的业务事件 ID，盒子用于日志；`detect.startTime/endTime` 是毫秒时间戳。

**discovery.enabled**
`config.yml` 字段。mDNS 广播总开关。**多盒子部署必须 true**。注意是 `enabled`（带 d）跟其它段不一样。

**Discovery** （mDNS Service Discovery）
[reference/04-key-concepts.md §11](../reference/04-key-concepts.md)。盒子启动时通过 zeroconf.RegisterProxy 广播 `_smartaudio._tcp.local.` 服务，让客户端零配置发现。

**dns-sd**
macOS 自带的 mDNS 命令行工具。`dns-sd -B _smartaudio._tcp` 列服务，`dns-sd -L <instance>` 看详情。

**dummy-card**
rk3308 上的 I2S 输入对应的 ALSA 卡名（`hw:1,0`）。在 `smartaudio devices` 输出里识别它，对应的 index 写入 `audio.device_index`。

---

## E

**EBUSY**
Linux errno 16, "Device or resource busy"。在 net_audio 里常见于 GPIO sysfs 非幂等导致 `echo X > /sys/class/gpio/export` 第二次失败（详见 [scenarios/02-sensitivity-tuning.md](../scenarios/02-sensitivity-tuning.md)）。

**embed (go:embed)**
Go 1.16+ 的内置功能，编译时把静态资源打包进 binary。admin 用 `go:embed all:static` 把 Vue3 dist 打进 binary，**单 binary 部署**。

---

## F

**fake-hwclock**
Linux 软件假时钟。Rock Pi S 无 RTC，靠 fake-hwclock 关机时存盘 + 开机恢复。**断电硬关机会丢时间**。

**fileUploader**
internal/upload/file_uploader.go 中的后台上传器。`intervalUpload()` 每 `Upload.Interval` ms 扫一次 cacheFileStream。

---

## G

**Gin**
Go HTTP 框架。两个 Go 进程的 HTTP/WebSocket 路由都基于 Gin。

**GPIO** (sysfs)
Linux 通过 `/sys/class/gpio/` 控制 GPIO 引脚。SmartAudio 用 GPIO 控 TPL0501 数字电位器写灵敏度。

---

## I

**I2S** (Inter-IC Sound)
数字音频接口协议。Rock Pi S 通过 I2S 接收麦克风阵列板的 4 通道 PCM。

**internal/channel/label.go**
A/B/C/D 物理标签 ↔ 内部声道号的权威映射代码。

**Input overflowed**
PortAudio 错误。采集回调没及时取走 ALSA 数据。**单台高频出现 = 麦克风采集硬件个体问题**（详见 agent 记忆里的 `box_audio_overflow_hardware.md`）。

---

## J

**journalctl**
systemd 日志查询。`journalctl -u SmartAudio -f` 实时跟主程序日志。

**JWT** (JSON Web Token)
admin 的鉴权方式（HS256）。默认 secret 是 `admin-config.yml` 里的固定字符串（**安全债，出厂必修**）。

---

## L

**legacy 录音路径**
指 `/api/v1/start_record` + `/upload_audio` + `/stop_record` 这套老接口 + cacheFileStream + interval upload 的链路。新接入推荐 stop_and_fetch，不用这套。

---

## M

**MAC**
网卡硬件地址。**每台 Rock Pi S 唯一**，是当前唯一可靠区分多盒子的硬件标识（其它 hostname / SN / IP / machine-id 都克隆撞库）。但业务层拿不到（要 ARP 扫描）。

**machine-id**
Linux 系统标识 (`/etc/machine-id`)。出厂镜像克隆未重置 → 多盒子撞同一份 → DHCP DUID / 某些 systemd 机制会撞。

**mDNS** (Multicast DNS)
RFC 6762。零配置局域网服务发现协议。走 UDP/5353 组播。详见 Discovery 条目。

**multipart/form-data**
HTTP 多部分响应格式。stop_and_fetch 多通道响应用这个：每个 part 一个通道 WAV，`Content-Disposition` 含 `channel=N`。

---

## N

**netplan**
Debian/Ubuntu 网络配置框架。admin 的 `/api/network` 接口操作 `/etc/netplan/01-ethernet.yaml`。

**ntpdate**
经典 NTP 客户端。`ntpdate ntp.aliyun.com` 校时间。Rock Pi S 无 RTC 必须配合 `fake-hwclock save`。

**narb** (`.narb`)
RingBuffer 文件后缀（recordings.narb）。无特殊含义，是项目内约定。**绝对不要打进备份 tar**（实时追加写）。

---

## P

**PCM** (Pulse-Code Modulation)
原始数字音频样本格式。BufferPool 里存的就是 PCM。

**PortAudio**
跨平台音频 I/O 库。主程序通过 PortAudio 访问 ALSA。需要 CGO。

---

## R

**RegisterProxy**
zeroconf 库的方法，允许显式指定 mDNS host 名（不依赖系统 hostname）。net_audio 用它解决多盒子 hostname 撞 `rockpis.local.` 问题。

**recordings.narb**
默认的 RingBuffer 文件路径（`/home/rock/audio/recordings.narb`）。10GB 大小，覆盖式写。

**RingBuffer**
[reference/04-key-concepts.md §3](../reference/04-key-concepts.md)。磁盘上的环形音频文件，覆盖式写入。当 `storage.enable=true` 时启用。代码位置：`internal/storage/ringbuffer.go`。

**Rock Pi S**
当前的硬件载板。rk3308 SoC（4×Cortex-A35）+ 512MB RAM + 板载 I2S。**无 RTC**。

**rockpis**
出厂镜像默认的系统 hostname。所有盒子都叫这个 → mDNS 撞库 → 用 RegisterProxy 在 mDNS 层绕开。

---

## S

**sensitivities** / **sensitivity**
4 通道灵敏度。0-255 整数，控制 TPL0501 数字电位器，影响麦克风采集前端增益。

**sensitivities_by_label**
新版灵敏度接口入参格式：`{"A": 180, "B": 200, ...}`。跟 `sensitivities` 数组互斥。

**SmartAudio**
产品命名 / 主程序名 / systemd unit 名（`SmartAudio.service`）。仓库根目录的 Go module 也叫 `smartaudio`（小写）。

**SmartAudio Admin** / **smartaudio-admin**
管理后台进程。仓库 `admin/` 子目录的独立 Go module。systemd unit 是 `SmartAudioAdmin.service`。

**SmartPLC**
典型的上游算法集成方（产品名）。**不是 net_audio 的一部分**，是用 net_audio 的客户系统之一。

**SN** (Serial Number)
设备序列号。**出厂镜像克隆未重置**，多盒子撞库。靠 SN 区分设备的逻辑在 SN 撞库时会失效，出厂烧录必须重置。

**SoC** (System on Chip)
Rock Pi S 上的 rk3308 芯片，含 4 核 Cortex-A35 + I2S/SPI/GPIO 等外设。

**stage** / **detectStageID**
见 detect 条目。

**stop_and_fetch**
**核心算法直调接口** `POST /api/v1/stop_and_fetch`。按时间窗从 BufferPool 取音频字节流。"不 stop，只 fetch"——名字保留了历史语义，实际只读不停。

**storage.enable**
`config.yml` 字段。RingBuffer 总开关。true = 落盘 recordings.narb；false = 纯内存模式。

**stream / WebSocket stream**
实时音频流接口 `/api/v1/stream/live`。客户端订阅通道，盒子持续推送 PCM/WAV chunk。

**systemd-machine-id-setup**
重置 machine-id 的命令。出厂烧录环节应该跑。

---

## T

**TPL0501**
数字电位器芯片型号。通过 SPI（用 GPIO sysfs 模拟）写入 0-255 设置阻值，控制麦克风采集前端增益。

**TPL0501-V1.4.sh**
盒子上的灵敏度脚本 (`/home/rock/audio/TPL0501-V1.4.sh`)。主程序启动时调用，给 4 通道写 yaml 里配的 sensitivities 值。

**time_sync**
强前置接口 `POST /api/v1/time_sync`。客户端报上自己的当前时间戳，盒子记下偏移。后续按时间戳的接口（如 stop_and_fetch）按此偏移换算。

---

## U

**Upload.Enable**
`config.yml` 的 `upload.enable` 字段。控制 legacy cacheFileStream 后台上传。跟 stop_and_fetch 路径独立。

**Upload.Endpoint**
后台上传的目标 URL。当 `Upload.Enable=true` 时，cacheFileStream 里的 Wait 文件 POST 到这里。

---

## V

**Vite**
admin 前端打包工具。`npm run build` 输出到 `web/dist/` → 拷到 `internal/api/static/` → embed。

**Vue 3**
admin 前端框架。配 Element Plus + Pinia + Vue Router 4。

---

## W

**WAV**
音频文件格式。stop_and_fetch 单通道响应直接返裸 WAV 字节流。盒子内部由 `utils.CreateWavBytes` 把 PCM + sampleRate + bitDepth 编码成完整 WAV（含 RIFF 头）。

**WebSocket**
持久全双工连接协议。stream/live 和调试 stream 用 WebSocket 推音频。

---

## Z

**zeroconf**
Go 的 mDNS 库 (`github.com/grandcat/zeroconf`)。详见 RegisterProxy / Discovery 条目。

---

## 配置字段速查

| 字段 | 段 | 默认 | 说明 |
|---|---|---|---|
| `audio.sample_rate` | audio | 44100 | 采样率 Hz |
| `audio.bits_per_sample` | audio | 16 | 量化位深 |
| `audio.channels` | audio | 4 | 物理通道数 |
| `audio.split_channels` | audio | 4 | 分发声道数 |
| `audio.device_index` | audio | — | PortAudio 设备索引（`smartaudio devices` 查） |
| `audio.sensitivities` | audio | [240]×4 | 4 通道灵敏度 |
| `buffer_pool.enable` | buffer_pool | true | BufferPool 总开关 |
| `buffer_pool.duration` | buffer_pool | 30 | 保留时长（秒） |
| `storage.enable` | storage | true | RingBuffer 总开关 |
| `storage.file` | storage | recordings.narb | RingBuffer 文件路径 |
| `storage.max_size` | storage | 10GB | RingBuffer 容量上限 |
| `upload.enable` | upload | false | legacy 上传总开关 |
| `upload.endpoint` | upload | "" | 后台 POST 目标 URL |
| `upload.interval` | upload | 1000 | 扫描周期 ms |
| `discovery.enabled` | discovery | true | mDNS 广播开关 |
| `discovery.api_port` | discovery | 8090 | 广播给客户端的端口 |
| `system.sensitivity_script` | system | TPL0501-V1.4.sh | 灵敏度脚本路径 |

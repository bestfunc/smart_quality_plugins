# 关键概念

这一份是 net_audio 里**容易让人绕晕**的核心概念。读完应该能听懂同事在 review 里说"BufferPool 切片"、"cacheFileStream 入队"、"label C 是 ch1"等术语的意思。

按"会被哪种角色用到"分组排序，从最常见的开始。

---

## 1. A / B / C / D 物理标签（极重要）

**盒子正面从左到右物理标签 A B C D，与内部声道号不是顺序对齐：**

| 物理位置（左→右） | A | B | C | D |
|---|---|---|---|---|
| 内部声道号 | **3** | **4** | **1** | **2** |

权威翻译代码：`internal/channel/label.go`（主程序）和 `admin/internal/service/channel_label.go`（admin 镜像）。

为什么不对齐：硬件接线时按方便走线的顺序连，没考虑"逻辑顺序与物理顺序一致"。改硬件接线代价太高，所以软件层做翻译。

**所有面向用户的接口同时支持两种入参形式**：
- 旧（按声道顺序）：`channels=[1,3]` / `sensitivities=[ch1,ch2,ch3,ch4]`
- 新（按物理标签，推荐）：`channel_labels=["C","A"]` / `sensitivities_by_label={"A","B","C","D"}`

任何新前端建议**只用 label**。讨论问题时如果对方说"3 号麦"，先问清楚是物理 3 还是内部 3。

---

## 2. BufferPool（内存音频环形缓冲）

**位置**：`internal/audio/buffer_pool.go`
**配置**：`buffer_pool` section，`enable` + `duration`（保留时长）+ 通道数

BufferPool 是**内存里的最近 N 秒音频**，按 channel 切片存储 PCM 样本，附时间戳。所有"按时间窗取音频"的请求都从这里取数据。

```
BufferPool.GetData(channels=[1,3], startTime, endTime)
  → map[channel]PCM bytes
```

它的关键属性：
- **时间窗 = 现在 - duration 到现在**。超出窗口外的请求 → 404 (`STAGE_NOT_FOUND`)
- **窗口左右截断**：请求窗超出 BufferPool 范围时，自动截到 BufferPool 边界
- **写满则覆盖最早**：固定容量，不无限增长
- **跟 RingBuffer 是两个东西**：BufferPool 在内存，RingBuffer 在磁盘

用法见 [reference/11-ai-integration.md](./11-ai-integration.md)。

---

## 3. RingBuffer（recordings.narb 磁盘环形）

**位置**：`internal/storage/ringbuffer.go`
**文件**：`/home/rock/audio/recordings.narb`（默认）
**配置**：`storage` section，`enable` + `file` + `max_size`（默认 10GB）

RingBuffer 是**磁盘上的环形音频文件**，达到 `max_size` 后从头覆盖。当 `storage.enable=false` 时纯内存模式，不落盘。

它的设计目的：**断电恢复后的音频回溯**。BufferPool 在内存里，重启就没了；RingBuffer 留在磁盘上，配合算法侧的事件时间戳能取到几小时甚至几天前的音频。

⚠️ **绝对不要把 recordings.narb 打进备份 tar** —— 文件是实时追加写的，tar 会一直追日志尾巴，几百 MB 起步。备份只打 binary + config + systemd unit。详见 agent 记忆里的 `feedback_deployment_style.md`。

---

## 4. cacheFileStream（legacy 录音队列）

**位置**：`internal/controller/cache_file.go`
**操作入口**：`internal/controller/controller.go` 里的 `Start/Stop/UpdateJson/GetFileHandle`

cacheFileStream 是 **legacy 录音路径**用的内存队列，按 channel 切片：

```
cacheFileStream[1]: []CacheFile{
  {Name, Path, WavData, UpdateState: Wait/Uploading/Uploaded, ...},
  ...
}
```

入队场景：客户端调 `/api/v1/start_record` 开始一段录音 → 期间音频写入 CacheFile → `/api/v1/stop_record` 或 `/upload_audio` 把 state 置 `Wait` → 后台 `upload.intervalUpload` 扫到 Wait → POST 到 `upload.endpoint`。

**算法直调接口 `/api/v1/stop_and_fetch` 不入这个队列**，它从 BufferPool 直取直返。两条路径完全独立。

---

## 5. detectStageID / stage（事件标识）

上游算法用 `detectStageID` 标识一次"事件"。每次 `stop_and_fetch` 请求都带：
- `detectStageID`: 算法侧自己的 ID 字符串（业务语义）
- `startTime` / `endTime`: 毫秒级时间戳

盒子用 stageID 仅用于**日志可读性**（错误日志会带 stage ID），不做状态保持。同一个 stageID 反复调可以拿到同一段音频（只要还在 BufferPool 窗口内）。

---

## 6. channel_labels（接口入参的物理标签）

所有新接口的入参支持物理标签：

```json
{
  "channel_labels": ["A", "C"],   // 物理位置
  "detect": {"detectStageID":"ev1","startTime":1700000000000,"endTime":1700000001000}
}
```

内部统一翻译成 `channels=[3,1]`。响应时除了 `channel: 3` 还回 `channel_label: "A"` 让上游对齐。

---

## 7. sensitivities_by_label（灵敏度的双轨入参）

灵敏度更新接口同时接受两种：

```json
// 形式 A（旧）
{"sensitivities": [120, 150, 180, 200]}

// 形式 B（新）
{"sensitivities_by_label": {"A":180, "B":200, "C":120, "D":150}}
```

互斥，二选一。响应里同时回两种视角。详见 [scenarios/02-sensitivity-tuning.md](../scenarios/02-sensitivity-tuning.md)。

---

## 8. 灵敏度（Sensitivity）

物理意义：**麦克风采集前端的模拟增益**，由 TPL0501 数字电位器控制（0-255）。值越大增益越大，麦克风越"灵"，远处声音也能录到，但**噪声底也跟着抬**。

每个通道独立配置一个值。**配置以 yaml 里 `audio.sensitivities` 为权威源**，启动时由主程序读取并调用 `system.sensitivity_script` 写到 TPL0501。

⚠️ **更新灵敏度必须重启 SmartAudio**（v3.5.3 路径）：
1. admin 写 yaml
2. exec `/home/rock/audio/restart_smartaudio.sh`
3. `systemctl stop SmartAudio` → sleep 2s → `start` → 轮询 is-active

历史尝试过"热更新 TPL0501 脚本不重启服务"路径（v3.5.2），但 GPIO sysfs 非幂等导致 EBUSY，已在 8066c9e 回退。详见 [scenarios/02-sensitivity-tuning.md](../scenarios/02-sensitivity-tuning.md)。

---

## 9. Time Sync（时间同步前置）

**位置**：`internal/timesync/time_sync.go`
**接口**：`POST /api/v1/time_sync`

所有跟"时间窗"相关的接口（`upload_audio`、`stop_and_fetch`）**必须**前置一次时间同步。客户端首次调用前 POST 一次 `/time_sync`，盒子记下客户端时钟与盒子时钟的偏移；后续按时间窗的请求会按这个偏移换算。

不同步直接调 `stop_and_fetch` → 400 `"time synchronization required"`。

为什么强制：盒子无 RTC，每次启动时间可能错乱（实测某台盒子停在 2024-08-01）。如果客户端用自己的时钟传时间窗，必须先告诉盒子怎么换算。

---

## 10. 双进程 + 谁能改什么

- **改音频采集 / stream / mDNS / 算法接口** → 改主程序（仓库根 module `smartaudio`）
- **改鉴权 / Web UI / 配置变更 / 网络 IP / 灵敏度入口** → 改 admin（`admin/` module `smartaudio-admin`）
- admin 不直接操作硬件，而是通过 systemctl 重启主程序、或 exec shell 脚本（restart_smartaudio.sh / TPL0501-V1.4.sh）

---

## 11. mDNS 与"为什么用 RegisterProxy"

普通 `zeroconf.Register` 用系统 hostname 做 mDNS A 记录。出厂镜像所有 Rock Pi S hostname 都是 `rockpis` → 多盒子同网段全部撞 `rockpis.local.`，客户端只能看到第一个回包的盒子。

v3.5 改用 `zeroconf.RegisterProxy` 显式指定 host = `smartaudio-<SN>.local.`（每台 SN 不同），解决了撞库。**这是当前架构下区分多盒子的唯一可靠手段** —— SN / IP / hostname / machine-id 都克隆了，只有 MAC 真随机，但客户端拿不到 MAC。

---

## 12. Upload.Enable（legacy 上传开关）

`config.yml` 里 `upload.enable` 控制 **legacy 录音队列**的后台上传。

- `enable=true`：每 `interval` ms 扫一次 cacheFileStream，把 state=Wait 的 POST 到 `upload.endpoint`
- `enable=false`：ticker 仍然跑，但不发请求

**它跟 `/api/v1/stop_and_fetch` 没关系** —— 算法直调路径不入 cacheFileStream，不受 enable 影响。

历史决策：如果现场只走 stop_and_fetch，`upload.enable` 可以设 false 避免误传。

---

## 13. Discovery.Enabled（mDNS 总开关）

`config.yml` 里 `discovery.enabled` 控制 mDNS 广播。

- `true`（默认）：盒子启动时注册 mDNS，客户端可零配置发现
- `false`：盒子不广播，客户端必须事先知道 IP

⚠️ 多盒子部署强烈建议 **保持 true**，否则没法区分不同盒子。**字段名是 `enabled`（带 d）**，跟其它 section 的 `enable` 不一样 —— 这是历史 yaml 设计不统一留下的小坑，改不了已部署配置。

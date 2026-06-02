# 上游算法 / 系统集成

> 这一份是给**上游算法 / PLC / 业务后台开发者**看的。
>
> 完整对接文档另见：`docs/API_HARDWARE.md` 和 `docs/API_SENSITIVITY.md`。

## 4 条可用的接入路径

| 路径 | 协议 | 用途 | 谁该用 |
|---|---|---|---|
| `POST /api/v1/stop_and_fetch` | HTTP | **算法直调取音频字节**（核心） | 算法节点 / PLC / 业务后台 |
| `POST /api/v1/start_record` + `/upload_audio` + `/stop_record` | HTTP | Legacy 录音 + 异步上传 web | 老系统兼容 |
| `WS /api/v1/stream/live` | WebSocket | 订阅实时 PCM 流 | 实时分析 / 监控大屏 |
| mDNS `_smartaudio._tcp` | UDP/5353 | 零配置发现盒子 | 任何接入方（找 IP 用） |

**新接入推荐 `stop_and_fetch` + mDNS 发现**。下面详细说每一条。

---

## 路径 1：算法直调（stop_and_fetch）—— 最常用

### 协议摘要

```
POST http://<盒子IP>:8090/api/v1/stop_and_fetch
Content-Type: application/json

{
  "channels": [1, 3],                           // 内部声道号，老格式
  // 或：
  "channel_labels": ["A", "C"],                 // 物理标签，新格式（推荐）
  "detect": {
    "detectStageID": "ev_20260528_001",         // 算法侧业务 ID
    "startTime": 1748395620000,                 // 毫秒时间戳
    "endTime": 1748395622000
  },
  "return_audio": true                          // false 时只确认请求送达，不返数据
}
```

### 响应格式（按通道数自动切换）

**单通道**（`channels.length == 1`）：

```
HTTP/1.1 200 OK
Content-Type: audio/wav
X-File-Md5: <hex>
X-Duration: 2.00
X-File-Name: channel_1_20260528160702.wav
X-Channel: 1
X-Channel-Label: C

<WAV 字节流>
```

**多通道**（`channels.length >= 2`）：

```
HTTP/1.1 200 OK
Content-Type: multipart/form-data; boundary=<boundary>

--<boundary>
Content-Disposition: form-data; name="audio"; filename="..."; channel=1; channel_label=C
Content-Type: audio/wav

<WAV 字节流>
--<boundary>
Content-Disposition: form-data; ... channel=3; channel_label=A
...
```

### 强前置：先调 time_sync

**不同步直接调 stop_and_fetch → HTTP 400 `"time synchronization required"`**。

```
POST http://<盒子IP>:8090/api/v1/time_sync
Content-Type: application/json
{ "client_time": <客户端当前毫秒时间戳> }
```

盒子记录客户端时钟偏移。同步一次足够（除非盒子重启）。

详细见 `docs/TIME_SYNC.md`。

### 错误码

| HTTP | error_code | 含义 |
|---|---|---|
| 400 | — | body 解析失败 / `startTime >= endTime` / `channels` + `channel_labels` 同时传 |
| 400 | `time synchronization required` | 没调过 `/api/v1/time_sync` |
| 404 | `STAGE_NOT_FOUND` | 时间窗完全在 BufferPool 范围外（过去太久或未来） |
| 404 | `NO_AUDIO_DATA` | 时间窗在 BufferPool 范围内，但请求的通道无数据 |
| 503 | — | BufferPool 未初始化 / 未启用 |

### 完整工作流

```python
# 伪代码
# 1. mDNS 找盒子
box_ip = mdns_resolve("smartaudio-<sn>._smartaudio._tcp")

# 2. 时间同步（首次 + 盒子重启后）
requests.post(f"http://{box_ip}:8090/api/v1/time_sync",
              json={"client_time": int(time.time()*1000)})

# 3. 算法触发事件，拿音频
events = monitor_for_events()
for ev in events:
    r = requests.post(f"http://{box_ip}:8090/api/v1/stop_and_fetch",
                      json={
                          "channel_labels": ["A", "C"],
                          "detect": {
                              "detectStageID": ev.id,
                              "startTime": ev.start_ms,
                              "endTime": ev.end_ms
                          },
                          "return_audio": True
                      })
    if r.status_code == 200:
        if r.headers["Content-Type"] == "audio/wav":
            wav_bytes = r.content
        else:  # multipart
            wav_by_channel = parse_multipart(r.content)
    elif r.status_code == 404 and "STAGE_NOT_FOUND" in r.text:
        # 太晚了，BufferPool 里已经覆盖掉了
        pass
```

完整场景：[scenarios/03-upstream-algorithm-integration.md](../scenarios/03-upstream-algorithm-integration.md)。

---

## 路径 2：Legacy 录音 + upload

适合**已经接好老系统**的客户，不建议新接入用。

```
POST /api/v1/start_record  → 开始录音
POST /api/v1/upload_audio  → 提交一段音频（音频数据 base64 在 body）
POST /api/v1/stop_record   → 结束录音
```

录完的文件入 cacheFileStream，后台 ticker 按 `upload.interval` 扫，`upload.enable=true` 时 POST 到 `upload.endpoint`。

**注意**：跟 `stop_and_fetch` 路径**完全独立**，不会双发。但如果两边都在用，同一段音频可能被分别发到两个不同的 web。详见 [reference/04-key-concepts.md §12](./04-key-concepts.md)。

---

## 路径 3：WebSocket Stream

```
WS ws://<盒子IP>:8090/api/v1/stream/live
```

客户端先发订阅消息：

```json
{ "action": "subscribe", "channels": [1, 2, 3, 4],
  "quality": { "sample_rate": 44100, "bit_depth": 16 } }
```

之后盒子持续推送 audio_batch 消息：

```json
{ "type": "audio_batch",
  "channels": {
    "1": "<base64 PCM>",
    "2": "<base64 PCM>",
    ...
  },
  "timestamp": 1748395620123 }
```

适用：监控大屏波形 / dBA 实时显示 / 实时算法（要求带宽 + 网络稳定）。

会话管理：

- `GET /api/v1/stream/sessions` —— 看当前所有 stream 会话
- `DELETE /api/v1/stream/session/:id` —— 强制踢一个会话

---

## 路径 4：mDNS 发现

盒子启动时广播：

```
服务类型：_smartaudio._tcp.local.
实例名：  SmartAudio-<SN>
主机名：  smartaudio-<SN>.local.
端口：    8090
TXT：     sn=<SN> model=<model> fw=v3.5.3
```

客户端发现：

```bash
# macOS
dns-sd -B _smartaudio._tcp
dns-sd -L SmartAudio-<sn> _smartaudio._tcp local

# Linux (Avahi)
avahi-browse -rt _smartaudio._tcp

# Go (zeroconf)
resolver, _ := zeroconf.NewResolver(nil)
entries := make(chan *zeroconf.ServiceEntry)
go resolver.Browse(ctx, "_smartaudio._tcp", "local.", entries)
for e := range entries {
    // e.AddrIPv4 是盒子 IP
}
```

**admin 端口（8091）没单独广播**，约定：`api_port + 1 = admin 端口`。如果将来 admin 端口可配置，会增加 TXT 字段。

---

## 鉴权模型

### 主程序 8090

- **`/api/v1/stop_and_fetch`** — **无鉴权**（v3.5.x 决策，便利性优先；前提是产线内网封闭）
- **`/api/v1/time_sync`** — 无鉴权
- **`/api/v1/start_record` / `/upload_audio`** — RSA 签名 token (X-Token 头)，由集成方持有私钥
- **`/api/v1/stream/live`** — 可选鉴权（按配置）

### Admin 8091

- **`/api/audio/sensitivities` / `/api/network`** — **无鉴权**（v3.5.3 决策）
- 其它 admin 接口 — JWT (Bearer token)，登录后获取

详见 [reference/02-architecture.md §关键 API 路由](./02-architecture.md)。

---

## 内置工具：admin 调试

如果你是上游集成方，**不想自己写 WebSocket 客户端**也能看实时音频，admin Web UI 自带调试页面：

- 浏览器打开 `http://<盒子IP>:8091`
- 登录（出厂默认账号 / 密码见交付清单 —— 客户首次部署后应改密）
- 进 "调试工具" 页面 → 订阅通道 → 看实时波形 + dBA

这个 UI 走的是 `/api/debug/stream` WebSocket（admin 反向代理到主程序 8090 的 `/api/v1/stream/live`）。

---

## 常见错误处理建议

| 错误 | 应对 |
|---|---|
| `time synchronization required` | 调一次 `/api/v1/time_sync` 即可。**盒子重启后会丢同步状态**，需要重新调 |
| `STAGE_NOT_FOUND` | 时间窗太老（BufferPool 覆盖了）。算法侧要么提前调用 `start_record` 把音频锁进 cacheFileStream，要么调大 `buffer_pool.duration` |
| `NO_AUDIO_DATA` for some channel | 通道索引有错（参考 [reference/04-key-concepts.md §A/B/C/D 物理标签](./04-key-concepts.md)）。建议改用 `channel_labels` 入参避免搞错 |
| 连接超时 | 先 `ping <盒子IP>`，再 `curl http://<盒子IP>:8091/api/auth/login -X POST`。如果 8091 通 8090 不通，主程序挂了，看 `journalctl -u SmartAudio` |
| `Content-Type: multipart/form-data` 解析失败 | 用标准库（Go `mime/multipart`、Python `email.parser` 或 `requests-toolbelt`），不要手写 boundary 解析 |

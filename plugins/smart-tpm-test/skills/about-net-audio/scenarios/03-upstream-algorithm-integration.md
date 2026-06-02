# 场景：上游算法 / PLC 集成对接

## 你在干嘛

你是算法节点 / SmartPLC / 业务后台，要从盒子拉音频字节做自己的分析。

## 整体流程

```
1. mDNS 发现盒子（拿 IP）
   ↓
2. 调一次 /api/v1/time_sync（强前置）
   ↓
3. 业务触发事件 → 拿到 [startTime, endTime] 时间窗
   ↓
4. POST /api/v1/stop_and_fetch（拿 WAV 字节）
   ↓
5. 自己处理（推理、入库、转发都行）
```

## 第 1 步：mDNS 发现

如果你已经知道盒子 IP，跳过。否则用 mDNS：

### 用 Go (zeroconf)

```go
import "github.com/grandcat/zeroconf"

resolver, _ := zeroconf.NewResolver(nil)
entries := make(chan *zeroconf.ServiceEntry)
go resolver.Browse(ctx, "_smartaudio._tcp", "local.", entries)
for e := range entries {
    if len(e.AddrIPv4) > 0 {
        boxIP := e.AddrIPv4[0].String()
        sn := ""
        for _, txt := range e.Text {
            if strings.HasPrefix(txt, "sn=") {
                sn = strings.TrimPrefix(txt, "sn=")
            }
        }
        fmt.Printf("found box: ip=%s sn=%s\n", boxIP, sn)
    }
}
```

### 用 Python (zeroconf)

```python
from zeroconf import Zeroconf, ServiceBrowser

class Listener:
    def add_service(self, zc, type_, name):
        info = zc.get_service_info(type_, name)
        ip = ".".join(str(b) for b in info.addresses[0])
        sn = info.properties.get(b'sn', b'').decode()
        print(f"found box: ip={ip} sn={sn}")

zc = Zeroconf()
ServiceBrowser(zc, "_smartaudio._tcp.local.", Listener())
```

### 用命令行（调试）

```bash
# macOS
dns-sd -B _smartaudio._tcp
dns-sd -L SmartAudio-<sn> _smartaudio._tcp local

# Linux
avahi-browse -rt _smartaudio._tcp
```

每条返回里都有 `sn / model / fw` 字段。

## 第 2 步：时间同步

**这一步必做，否则 stop_and_fetch 返 400**。

```python
import time, requests

resp = requests.post(
    f"http://{box_ip}:8090/api/v1/time_sync",
    json={"client_time": int(time.time() * 1000)}
)
# 返回 server_time / received_at / processed_at / network_delay / time_offset
print(resp.json())
```

盒子记下你的客户端时钟相对盒子时钟的偏移。后续按时间戳的请求按此偏移换算。

**什么时候要重新同步**：
- 你的客户端时钟跳变（NTP 校准、时区切换）
- 盒子重启（盒子内的同步状态会丢）
- 长时间未通信（建议每小时或每个会话首次重新同步）

## 第 3 步：业务触发 → 拿时间窗

业务事件类型：
- PLC 接到产线传感器信号 → 当时秒级时间戳
- 算法节点检测到异常 → 当时毫秒级时间戳
- 定时任务：每分钟取一次最近 30 秒

时间窗应该 **完全落在 BufferPool 的保留时长内**（默认 30 秒）。如果业务延迟太大（比如算法侧攒了一批事件 1 分钟后才回头拉），可能 `STAGE_NOT_FOUND`。

## 第 4 步：拉音频

### 最简单的：单通道

```python
resp = requests.post(
    f"http://{box_ip}:8090/api/v1/stop_and_fetch",
    json={
        "channel_labels": ["A"],          # 只要物理 A 通道
        "detect": {
            "detectStageID": "ev_001",
            "startTime": 1748395620000,   # 毫秒
            "endTime": 1748395622000
        },
        "return_audio": True
    }
)

if resp.status_code == 200:
    assert resp.headers["Content-Type"] == "audio/wav"
    wav_bytes = resp.content
    md5 = resp.headers["X-File-Md5"]
    duration = float(resp.headers["X-Duration"])
    label = resp.headers["X-Channel-Label"]   # "A"
    # 存盘或喂给模型
    with open("event.wav", "wb") as f:
        f.write(wav_bytes)
elif resp.status_code == 404:
    err = resp.json()
    print(f"取数失败: {err['error_code']} - {err['message']}")
elif resp.status_code == 400:
    print(f"请求异常: {resp.text}")
```

### 多通道

```python
resp = requests.post(
    f"http://{box_ip}:8090/api/v1/stop_and_fetch",
    json={
        "channel_labels": ["A", "B", "C", "D"],   # 全部 4 通道
        "detect": {"detectStageID":"ev_002","startTime":..,"endTime":..},
        "return_audio": True
    }
)
# Content-Type: multipart/form-data; boundary=...
# 用 email.parser 或 requests-toolbelt 解析
```

#### 解析 multipart（Python 推荐方式）

```python
from requests_toolbelt.multipart import decoder

multipart_data = decoder.MultipartDecoder(resp.content, resp.headers["Content-Type"])

wav_by_channel = {}
for part in multipart_data.parts:
    cd = part.headers[b"Content-Disposition"].decode()
    # cd 形如 'form-data; name="audio"; filename="..."; channel=1; channel_label=C'
    # 拿 channel_label
    for kv in cd.split(";"):
        kv = kv.strip()
        if kv.startswith("channel_label="):
            label = kv.split("=")[1].strip('"')
            wav_by_channel[label] = part.content
```

#### Go 解析

```go
import "mime"
import "mime/multipart"
import "io"

_, params, _ := mime.ParseMediaType(resp.Header.Get("Content-Type"))
mr := multipart.NewReader(resp.Body, params["boundary"])
for {
    part, err := mr.NextPart()
    if err == io.EOF { break }
    // part.Header.Get("Content-Disposition") 拿到 channel / channel_label
    data, _ := io.ReadAll(part)
    // 处理
}
```

## 完整 Python 示例（生产可参考）

```python
import time
import requests
from zeroconf import Zeroconf, ServiceBrowser
from threading import Event

class SmartAudioClient:
    def __init__(self, sn_target):
        self.sn = sn_target
        self.box_ip = None
        self.box_port = 8090
        self._discovered = Event()
        self._synced = False

    def discover(self, timeout=5):
        def add_service(zc, type_, name):
            info = zc.get_service_info(type_, name)
            sn = info.properties.get(b'sn', b'').decode()
            if sn == self.sn:
                self.box_ip = ".".join(str(b) for b in info.addresses[0])
                self.box_port = info.port
                self._discovered.set()

        class L: add_service = staticmethod(add_service); update_service = lambda *a: None; remove_service = lambda *a: None
        zc = Zeroconf()
        ServiceBrowser(zc, "_smartaudio._tcp.local.", L())
        self._discovered.wait(timeout)
        zc.close()
        if not self.box_ip:
            raise RuntimeError(f"box sn={self.sn} not found via mDNS")

    def sync_time(self):
        r = requests.post(
            f"http://{self.box_ip}:{self.box_port}/api/v1/time_sync",
            json={"client_time": int(time.time() * 1000)}
        )
        r.raise_for_status()
        self._synced = True

    def fetch(self, channels_labels, start_ms, end_ms, stage_id):
        if not self._synced:
            self.sync_time()
        r = requests.post(
            f"http://{self.box_ip}:{self.box_port}/api/v1/stop_and_fetch",
            json={
                "channel_labels": channels_labels,
                "detect": {
                    "detectStageID": stage_id,
                    "startTime": start_ms,
                    "endTime": end_ms
                },
                "return_audio": True
            }
        )
        if r.status_code != 200:
            return None
        if r.headers["Content-Type"].startswith("audio/wav"):
            return {channels_labels[0]: r.content}
        else:
            from requests_toolbelt.multipart import decoder
            md = decoder.MultipartDecoder(r.content, r.headers["Content-Type"])
            result = {}
            for part in md.parts:
                cd = part.headers[b"Content-Disposition"].decode()
                label = next((kv.split("=")[1].strip('"')
                             for kv in cd.split(";")
                             if kv.strip().startswith("channel_label=")), None)
                if label:
                    result[label] = part.content
            return result

# 用法
client = SmartAudioClient(sn_target="<SN>")
client.discover()
client.sync_time()

# 事件循环
while True:
    ev = wait_for_event()
    wavs = client.fetch(
        channels_labels=["A", "B", "C", "D"],
        start_ms=ev.start_ms,
        end_ms=ev.end_ms,
        stage_id=ev.id
    )
    if wavs:
        process_event(ev, wavs)
```

## 集成测试建议

接入第一天先跑通这些：

1. ✅ mDNS 能发现盒子
2. ✅ `time_sync` 返回包含 `server_time`
3. ✅ `stop_and_fetch` 用一个 5 秒前的时间窗能拿到 WAV
4. ✅ 单通道 + 多通道两种响应格式都能解析
5. ✅ 取一个**远古时间窗**（10 分钟前），确认能正确收到 `STAGE_NOT_FOUND`
6. ✅ 不调 `time_sync` 直接 `stop_and_fetch`，确认收到 400 `time synchronization required`
7. ✅ 盒子重启 + 客户端继续工作，确认客户端能感知"需要重新 time_sync"

## 性能基线参考

- **stop_and_fetch 响应延迟**：单通道 2 秒窗 → 通常 < 100ms（内存切片 + WAV 编码）
- **多通道响应大小**：4 通道 × 2 秒 × 44.1kHz × 2byte = ~700KB（multipart 略有开销）
- **BufferPool 时延**：从音频被麦克风采到能被 `stop_and_fetch` 拿到，延迟 < 10ms
- **time_sync 精度**：±10ms（受网络抖动影响，盒子和算法节点同一交换机会更准）

## 已知坑

- **盒子时钟漂移**：无 RTC，长时运行可能漂秒级。如果算法需要严格时间对齐，要么开 ntpd 让盒子持续同步，要么定期重新 `time_sync`
- **撞 IP**：多盒子同网段抢同一静态 IP。**只信 mDNS 返回的 IP，不要在客户端硬编**
- **upload.enable 影响**：跟你这条路径无关。即使盒子的 `upload.enable=true` 配着，你的 `stop_and_fetch` 不会被它影响。详见 [reference/04-key-concepts.md §12](../reference/04-key-concepts.md)

# FAQ：上游集成方常见问题

## Q1：为什么我调 stop_and_fetch 返回 400 "time synchronization required"？

**A**：你必须先调一次 `POST /api/v1/time_sync`。这是强前置 —— 盒子需要知道你的客户端时钟相对盒子时钟的偏移，才能正确解释你传的 `startTime/endTime`。

每个客户端实例首次调用前同步一次。盒子重启后状态会丢，要重新同步。

详见 [reference/04-key-concepts.md §Time Sync](../reference/04-key-concepts.md)。

---

## Q2：channels 和 channel_labels 哪个推荐？

**A**：**新接入只用 channel_labels**。理由：

- 物理标签 A/B/C/D 跟实物对应，跟客户讨论时不会搞错
- 内部声道号 1/2/3/4 跟物理位置**不顺序对齐**（A=ch3 B=ch4 C=ch1 D=ch2），用错就拿错通道音频
- 接口同时支持，response 也同时返回两种视角，迁移成本零

老接入维持 channels 也能用，向后兼容。

---

## Q3：单通道和多通道响应为什么格式不一样？

**A**：单通道直接返裸 WAV 字节流，最简单可流式播放；多通道用 multipart/form-data 才能在一个响应里塞多个独立 WAV。

```
单通道 → Content-Type: audio/wav + 元信息在 header
多通道 → Content-Type: multipart/form-data; boundary=...
```

如果你不想分情况处理，传 `channel_labels=["A"]` 也能拿到 multipart 格式 —— 但浪费。

---

## Q4：multipart 怎么解析？

**A**：用语言标准库或主流多部分解析库，**不要手撕 boundary**。

- Python：`requests-toolbelt.multipart.decoder.MultipartDecoder`
- Go：`mime/multipart.NewReader`
- Java：Apache HttpClient `MultipartEntity` 反向解析 / Spring 自带
- Node.js：`multiparty` / `busboy`

每个 part 的 `Content-Disposition` 头里有 `channel=N` 和 `channel_label=X`，按 label 索引拿对应通道。

完整示例见 [scenarios/03-upstream-algorithm-integration.md](../scenarios/03-upstream-algorithm-integration.md)。

---

## Q5：STAGE_NOT_FOUND 是什么情况？

**A**：你请求的时间窗**完全在 BufferPool 范围外**。两种原因：

1. **太早**（请求时间 < 当前 - buffer_pool.duration）：BufferPool 已经把那段覆盖了。默认窗 30 秒，超过 30 秒前的请求会 404。
2. **太晚**（请求时间 > 当前 + 少量裕度）：你传了"未来时间"，盒子没采到。

应对：
- 把算法触发到调 stop_and_fetch 的延迟压短
- 加大 `buffer_pool.duration` 配置（要平衡内存占用）
- 历史音频取数走 RingBuffer 接口（参考 admin 的 `/api/storage/entries`）

---

## Q6：NO_AUDIO_DATA 跟 STAGE_NOT_FOUND 有什么区别？

**A**：
- `STAGE_NOT_FOUND`：时间窗超出 BufferPool 范围
- `NO_AUDIO_DATA`：时间窗在范围内，但请求的**那个通道**没数据（通道未启用 / 通道索引错）

如果是后者，检查 `channel_labels` 是不是有效（必须 A/B/C/D 之一）。

---

## Q7：能不能不用 mDNS，直接连盒子 IP？

**A**：可以。如果你的部署能保证 IP 稳定（已经规划好的产线），直接 hard-code IP 没问题。

但**多盒子部署强烈建议走 mDNS**：
- 出厂镜像所有盒子静态 IP 都是同一个，靠 mDNS 区分
- 现场可能临时调整 IP，硬编会失败
- mDNS 返回里有 SN，能跟盒子物理身份对齐

---

## Q8：能不能并发调 stop_and_fetch？

**A**：可以，盒子 API 是无状态的，并发取数不会互相影响。但同一个时间窗的不同通道**最好在一次请求里要**（用 `channel_labels=["A","B","C","D"]`），减少网络往返。

---

## Q9：取数延迟太大，能更快吗？

**A**：当前 stop_and_fetch 典型延迟 < 100ms（单通道 2 秒窗）。如果你需要更低延迟：

- 用 WebSocket `/api/v1/stream/live` 持续订阅，自己缓存，事件触发时从客户端缓存取
- 改用 UDP 协议接 raw PCM（**当前没做**，路线图阶段）

延迟瓶颈一般在网络抖动，不在盒子本身。

---

## Q10：upload.enable 关掉会影响我吗？

**A**：**不会**。

`upload.enable` 控制的是 **legacy 录音队列**（start_record / upload_audio 入 cacheFileStream + 后台 ticker）。`stop_and_fetch` 走完全独立的 BufferPool 路径。

如果你只走 stop_and_fetch，**建议盒子端配置 `upload.enable: false`**，避免现场误调老接口时偷偷把音频发到别的 web。

---

## Q11：stop_and_fetch 需要鉴权吗？

**A**：v3.5.x **当前无鉴权**。产品决策：现场算法 / PLC 直接调用便利性优先。

前提是：本接口暴露在产线**内网**。绝对不要把 8090 端口暴露到客户公网。

如果客户要求做鉴权，需要内部讨论加什么模式（API key / JWT / mTLS）。

---

## Q12：我能拿到比 30 秒更早的历史音频吗？

**A**：可以，但走另一条路径 —— admin 的 `/api/storage/entries`，从 RingBuffer 取磁盘上的音频（默认存最近 10GB）。

但 admin 接口走 JWT 鉴权，不适合算法侧高频调用。当前这条路径主要给"调试 / 复现"用，不适合生产实时取数。

---

## Q13：盒子时钟漂移厉害怎么办？

**A**：Rock Pi S 无 RTC，断电后可能丢时间。两个应对方向：

1. **盒子侧**：配置开机自动 NTP 同步（systemd-timesyncd 或 chrony）
2. **客户端侧**：定期重新调 `time_sync`（建议每小时一次），让偏移持续校准

如果业务对**毫秒级时间对齐**有强要求，建议每次取数前先 `time_sync`。

---

## Q14：能不能在响应里加自定义 metadata？

**A**：当前不支持。响应只带：
- Headers: `X-File-Md5 / X-Duration / X-File-Name / X-Channel / X-Channel-Label`
- Body: WAV 字节

如果你需要在盒子里附加业务元数据，**路线图阶段**有"扩展元数据接口"的设想，但目前未承诺时间。临时方案：你的客户端在拿到 WAV 后自己加 metadata 包一层。

---

## Q15：盒子重启对我的会话有什么影响？

**A**：
- WebSocket stream 会断 → 你需要重连
- time_sync 状态丢失 → 重连后第一个 stop_and_fetch 会 400，重新 time_sync 即可
- BufferPool 清空 → 重启前的音频全部丢失（除非 storage.enable=true 落在 RingBuffer）

实践：客户端要做"重连 + 重新 time_sync"的逻辑，作为通用错误处理路径。

---

## Q16：跟普通的录音文件 API 比，stop_and_fetch 优势是什么？

**A**：
- **取数时机**：传统录音必须先 start → 等录完 → 下载，你不知道事件什么时候发生；stop_and_fetch 事后按时间窗取，事件触发什么时候都行（只要在 BufferPool 窗口内）
- **不入文件系统**：BufferPool 在内存，取数即返字节流，没有磁盘 I/O 开销
- **多通道一次取**：一次请求拿 4 通道 multipart
- **时间窗精确**：算法侧自己控制几毫秒到几秒的窗口

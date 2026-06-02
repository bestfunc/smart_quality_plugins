# FAQ：开发同事常见问题

> *(部分内容含安全债信息，标 *(内部参考，不对外发)* 的段落不要复制给客户)*

## Q1：为什么 admin 和主程序是两个进程不是一个？

**A**：
- 主程序生命周期跟"持续采集"绑定，重启 = 录音中断。希望少动它。
- 管理后台需要频繁迭代（Web UI 更新、配置管理 API 加字段）
- admin 需要 systemctl 重启主程序的能力，**双进程时它是 systemctl 调用方**，逻辑干净；单进程不可能"自己重启自己"

代价：双进程通信走 systemctl + shell 脚本，略糙但可控。

详见 [reference/02-architecture.md](../reference/02-architecture.md)。

---

## Q2：A/B/C/D 跟 ch1234 怎么记？

**A**：

| 物理（左→右） | A | B | C | D |
|---|---|---|---|---|
| 内部声道 | 3 | 4 | 1 | 2 |

记法："**CDAB → 1234**"（C 在最左对应 1）。

如果忘了直接看权威表 `internal/channel/label.go::labelToChannel`，不要自己脑补。

---

## Q3：为什么 channel 映射要做翻译？硬件不能改吗？

**A**：硬件接线时按"方便走线"的顺序连了通道 1234 到物理 A/B/C/D，没考虑逻辑顺序。改硬件接线代价远高于在软件层做翻译。所以引入 `internal/channel/label.go` 作为权威翻译表。

历史选择：早期版本只暴露 channels=[1,2,3,4]，导致接入方反复搞错。v3.5 引入 channel_labels 之后强烈推荐新接入只用 labels。

---

## Q4：为什么 BufferPool 和 RingBuffer 同时存在？

**A**：

| | BufferPool | RingBuffer |
|---|---|---|
| 位置 | 内存 | 磁盘 (recordings.narb) |
| 容量 | duration 秒（默认 30） | 字节数（默认 10GB） |
| 用途 | 实时算法切片 | 故障回溯 / 调试 |
| 失存场景 | 主程序重启即丢 | 重启不丢，只在写满后覆盖最早 |

stop_and_fetch 走 BufferPool（低延迟）；历史音频走 RingBuffer（admin `/api/storage`）。

---

## Q5：为什么灵敏度更新不走"热更新 TPL0501 脚本"路径了？

**A**：v3.5.2 曾用 admin 直接 exec TPL0501 脚本的热更新路径，**第二次 PUT 必失败**。

根因：`SET_TPL0501_FUNC-V1.4.sh` 的 GPIO 初始化非幂等：
```bash
echo $gpio > /sys/class/gpio/export
```
第二次执行 EBUSY（Device or resource busy）。脚本没做"已 export 则跳过"判断，也没 unexport。

v3.5.3 回退到"写 yaml + restart SmartAudio"路径（commit 8066c9e），主程序在重启时一次性 export GPIO，不会重复。代价是 5 秒中断。

根治方向：改脚本让 export 幂等，但优先级低。详见 [scenarios/02-sensitivity-tuning.md](../scenarios/02-sensitivity-tuning.md)。

---

## Q6：admin 的 sensitivity / network 接口为什么没鉴权？

**A**：v3.5.3 产品决策。理由：

- 现场算法 / PLC 需要频繁调用这两个接口，加 JWT 反而增加现场调试复杂度
- 假设设备只暴露在产线**内网**封闭网络，外部恶意访问可能性低
- 跟其它 admin 接口（走 JWT）的不一致是已知的，跟产品一起决定的

但**不可推广到其它接口**。其它写操作（启停服务、改配置全文）必须保留 JWT，否则风险太大。

详见 [reference/10-deployment-modes.md §安全模型](../reference/10-deployment-modes.md) *(内部参考)*。

---

## Q7：mDNS 为什么用 RegisterProxy 不用 Register？

**A**：普通 `zeroconf.Register` 用系统 hostname 注册 A 记录。出厂镜像所有盒子 hostname 都是 `rockpis` → 多盒子撞库。

RegisterProxy 允许显式指定 mDNS host 名，让我们用 `smartaudio-<SN>.local.` 这种唯一名字。这样：
- 不改系统 hostname（不动出厂镜像）
- 多盒子 mDNS 不撞
- 客户端能按 SN 找盒子

详见 [reference/04-key-concepts.md §11](../reference/04-key-concepts.md)。

---

## Q8：upload.enable=true 但用 stop_and_fetch 的客户会发生什么？

**A**：

两条路径完全独立：
- stop_and_fetch → BufferPool 直取直返，**不入 cacheFileStream**
- upload.enable=true 只扫 cacheFileStream

所以如果只用 stop_and_fetch，upload.enable=true 等于空转 ticker（不会"双发同一片音频"）。

**但**：如果现场还有别的客户端调老的 `start_record` / `upload_audio`，cacheFileStream 会被填充，那些音频会被 POST 到 upload.endpoint，**跟 stop_and_fetch 走的是不同的 web**。这种"同一片音频到达两个不同 web"的情况要避免就关掉 upload.enable。

详见 [reference/04-key-concepts.md §12](../reference/04-key-concepts.md)。

---

## Q9：interval upload 为什么不放在算法直调路径？

**A**：interval upload 是 legacy 设计的"定时把 cacheFileStream 里的音频 POST 给一个固定 web"。这套适合"录完即上传"模式，不适合"算法精准取数"。

新 stop_and_fetch 是同步请求-响应模型 —— 算法侧主动拉，不需要盒子推。两套设计目标不同，不应该合并。

---

## Q10：为什么 discovery 段用 `enabled` 不是 `enable`？

**A**：历史一致性问题。早期 yaml 设计的几个段是 `enable`（realtime / upload / report / storage 等），后期加 discovery 段时 `DiscoveryConfig.Enabled` 字段用了 `enabled` 标签。

```go
type DiscoveryConfig struct {
    Enabled bool `yaml:"enabled"`  // 这里
    APIPort int  `yaml:"api_port"`
}
```

改一致性代价：所有已部署 yaml 都要改。**收益小于风险**，保持不动。

知道这个坑就行：grep `enable` 时记得 `discovery` 段是 `enabled`。

---

## Q11：测试机是哪台？怎么连？

**A**：测试机参考详见 agent 记忆里的 `reference_test_device.md`（在 `~/.claude/projects/<project>/memory/` 下）。包含：
- 测试机 IP
- SSH 公钥 / 密码（**内部参考**）
- Docker 交叉编译命令

---

## Q12：怎么本地跑主程序？我没 Rock Pi S。

**A**：理论上可以在 macOS / Linux x86 上跑，因为 PortAudio 跨平台。但：

- 没有 4 通道 I2S 麦克风阵列，BufferPool 是空的，stop_and_fetch 取数会空
- TPL0501 灵敏度脚本是 ARM 专用，本地起不来
- mDNS 注册的 host 是 `smartaudio-<SN>.local.`，需要假 SN

**推荐做法**：本地写单元测试覆盖逻辑层，跑硬件相关链路时用测试机（参考上一条）。

---

## Q13：admin 的前端怎么调试？

**A**：

```bash
cd admin/web
npm install
npm run dev    # 起 vite dev server，默认 5173
# 浏览器打开 http://localhost:5173
# API 请求会代理到 ../../.env 配的 admin 后端
```

修改前端后跑 `npm run build` 把 dist 拷到 `internal/api/static/`，重新 `go build`。

---

## Q14：怎么打包发布包？

**A**：参考 `release/smartaudio_v3.5.3_20260528/` 目录结构。流程：
1. 改 `gen_version.sh` 里的 VERSION
2. 运行 `bash gen_version.sh` 写 version.txt
3. 跑 Docker 交叉编译主程序、admin
4. 组装目录（bin + scripts + config + 文档 + SHA256SUMS）
5. tar -czf

参考最新一份包的 OPS_MANUAL.md 抄一份新的，更新 CHANGELOG。

---

## Q15：内部安全债清单 *(内部参考，不对外发)*

详见 agent 记忆里的 `project_admin_security_debt.md`。摘要：
- 默认 JWT secret 客户共享
- admin 出厂默认账号 / 密码（具体凭据见出厂交付清单 *(内部参考)*）
- SSH root 出厂默认密码（同上）
- private_key.pem / public_key.pem 在仓库根
- 出厂烧录环节是清理这些债的地方

跟客户对话时**不要提**这些细节，统一口径是"商业部署前会做安全加固"。

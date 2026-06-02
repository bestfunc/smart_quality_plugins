---
name: about-net-audio
display_name: net_audio 联网录音系统 产品百科
description: net_audio 联网录音系统的方方面面 — 给开发同事、现场运维工程师、上游集成方（SmartPLC / 算法接入方）查阅。涵盖产品定位、双进程架构、4 通道物理映射、灵敏度调校、上游接口对接、现场部署模式、技术栈、运维基线、路线图。能根据业务需求给出使用建议。
user-invocable: true
---

# net_audio 联网录音系统 产品百科

这是 **net_audio 联网录音系统**（产品/服务命名 `SmartAudio`）的自助百科。无论你是**开发同事**、**现场运维工程师**、**上游算法/PLC 集成方**，还是只想**查个术语**，都能从这里查到全部产品知识，并根据业务场景给出合理建议。

不写代码 / 不调用工具。如果需要写代码，去用项目里的开发类 skill（`code-review`、`run`、`verify` 等），或直接读 `internal/` / `admin/` 源码。

---

## 我是谁？我从哪里看起？

### 🧑‍💻 我是开发同事，要改代码 / 排 bug / 加新接口

按这个顺序看：

1. **[reference/02-architecture.md](./reference/02-architecture.md)** — 双进程拓扑（smartaudio + smartaudio-admin）、模块清单、数据流。改任何东西之前先建立"我动的是哪一段"的全局观。
2. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — A/B/C/D 物理标签、BufferPool、RingBuffer、cacheFileStream 这些**容易绕晕**的关键概念，**先看懂再动代码**。
3. **[reference/09-tech-stack.md](./reference/09-tech-stack.md)** — Go 1.23 + PortAudio + Vue3 + zeroconf 等具体版本与依赖。
4. **[scenarios/02-sensitivity-tuning.md](./scenarios/02-sensitivity-tuning.md)** — 灵敏度链路是历史上踩坑最多的地方（GPIO busy / TPL0501 / 重启路径），改这条链路之前必读。
5. **[faq/02-developer-questions.md](./faq/02-developer-questions.md)** — 老开发遗留的"为什么这里这么写"的答案集合。

### 🛠️ 我是现场运维工程师，要部署 / 升级 / 排故

按这个顺序看：

1. **[scenarios/01-field-deployment.md](./scenarios/01-field-deployment.md)** — 现场部署最常见的两条路径（首次 / 升级），含备份铁律和回滚动作。
2. **[scenarios/04-troubleshooting.md](./scenarios/04-troubleshooting.md)** — "ping 通但 SSH 不上"、"两台盒子带宽差异巨大"等高频症状的快速诊断流程。
3. **[reference/10-deployment-modes.md](./reference/10-deployment-modes.md)** — 出厂镜像的克隆冲突、静态 IP 策略、mDNS 发现 —— 任何多盒子讨论先翻这条。
4. **[faq/03-ops-questions.md](./faq/03-ops-questions.md)** — 现场 FAQ：恢复时钟、改 IP、升级失败回滚……
5. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — 至少看懂 §"A/B/C/D 物理标签"，否则跟客户讨论"3 号麦"会出歧义。

### 🔌 我是上游集成方（SmartPLC / 算法 / 业务后台），要把音频接进我的系统

按这个顺序看：

1. **[scenarios/03-upstream-algorithm-integration.md](./scenarios/03-upstream-algorithm-integration.md)** — 完整接入工作流：发现 → 时间同步 → 拉取 stage 音频字节 → 自己处理。
2. **[reference/11-ai-integration.md](./reference/11-ai-integration.md)** — `POST /api/v1/stop_and_fetch` 协议详解、单/多通道响应差异、`channel_labels` 物理标签翻译。
3. **[reference/04-key-concepts.md](./reference/04-key-concepts.md)** — `time_sync` 强前置、`BufferPool` 时间窗口、A/B/C/D 标签映射 —— 接入前先确认理解这三件事。
4. **[faq/01-integration-questions.md](./faq/01-integration-questions.md)** — 上游集成方最常踩的坑（鉴权、时间偏移、多通道 multipart 解析）。

### 📚 我只是想查个术语 / 概念

直接 → **[glossary/terms.md](./glossary/terms.md)**

---

## 完整索引

### Reference（事实层）— `reference/`

| 文件 | 内容摘要 |
|---|---|
| [01-product-overview.md](./reference/01-product-overview.md) | 产品一句话定位、典型场景、与"普通录音机"的本质差异 |
| [02-architecture.md](./reference/02-architecture.md) | 双进程 + Web 前端 embed、核心模块表、音频采集→分发→落盘→上游消费的数据流 |
| [04-key-concepts.md](./reference/04-key-concepts.md) | 13 个核心概念（A/B/C/D 标签、BufferPool、RingBuffer、cacheFileStream、stage…） |
| [05-target-customers.md](./reference/05-target-customers.md) | 适配场景的工业/产线画像，"什么场景适合 / 不适合用 net_audio" |
| [07-pricing-business-model.md](./reference/07-pricing-business-model.md) | 商业交付模式：硬件 + 软件一体 / 软件单独授权 / OEM 集成（粗框架，<TODO> 由商务侧补） |
| [08-roadmap.md](./reference/08-roadmap.md) | v3.5 已交付清单 + 后续路线设想（**弱措辞**） |
| [09-tech-stack.md](./reference/09-tech-stack.md) | Go / PortAudio / Vue3 / zeroconf / TPL0501 / netplan / systemd 的版本和取舍 |
| [10-deployment-modes.md](./reference/10-deployment-modes.md) | 单盒子 / 多盒子产线 / 测试机；出厂镜像 4 大克隆冲突；静态 IP vs DHCP |
| [11-ai-integration.md](./reference/11-ai-integration.md) | 算法直调 (`stop_and_fetch`) + 录音/上传 + WebSocket stream + mDNS 发现 4 条接入路径 |

### Scenarios（场景层）— `scenarios/`

| 文件 | 内容摘要 |
|---|---|
| [01-field-deployment.md](./scenarios/01-field-deployment.md) | 现场首次部署 + 滚动升级 + 备份与回滚 |
| [02-sensitivity-tuning.md](./scenarios/02-sensitivity-tuning.md) | 灵敏度调校：A/B/C/D 4 通道独立、yaml+restart 路径、为什么不再用热更新 |
| [03-upstream-algorithm-integration.md](./scenarios/03-upstream-algorithm-integration.md) | SmartPLC / 自定义算法接入：发现 → 时间同步 → stage 拉音频 → 业务处理 |
| [04-troubleshooting.md](./scenarios/04-troubleshooting.md) | 8 类高频故障：连不上 / 带宽差 / 时钟丢 / overflow / yaml restart 失败 / mDNS 找不到 / 静态 IP 撞 / GPIO busy |

### FAQ（话术层）— `faq/`

| 文件 | 内容摘要 |
|---|---|
| [01-integration-questions.md](./faq/01-integration-questions.md) | 上游对接方 FAQ：鉴权、时间偏移、multipart 解析、错误码 |
| [02-developer-questions.md](./faq/02-developer-questions.md) | 开发 FAQ：模块/接口的设计原因、踩过的坑、为什么这里这么写 |
| [03-ops-questions.md](./faq/03-ops-questions.md) | 运维 FAQ：升级失败 / 时钟 / 静态 IP / mDNS / 公钥失效 |

### Glossary（术语层）— `glossary/`

| 文件 | 内容摘要 |
|---|---|
| [terms.md](./glossary/terms.md) | 全部专有名词、缩写、配置字段名解释 |

---

## 当你被问到的时候

> **开发同事**：「这个 channel_labels 跟内部声道是怎么映射的？」
> **你**：先看 [reference/04-key-concepts.md §A/B/C/D 物理标签](./reference/04-key-concepts.md)，里面有完整映射表（A=ch3 B=ch4 C=ch1 D=ch2）+ 权威翻译代码位置 (`internal/channel/label.go`)，可直接引用。

> **现场运维**：「客户说升级后盒子在 dns-sd 里找不到了。」
> **你**：先看 [scenarios/04-troubleshooting.md §mDNS 找不到盒子](./scenarios/04-troubleshooting.md)，95% 是 `discovery.enabled: false` 被误设；剩下 5% 是客户端 mDNS 缓存或者多播被路由禁。

> **上游集成方**：「我的 PLC 调 stop_and_fetch 返回 400 'time synchronization required'。」
> **你**：先看 [faq/01-integration-questions.md §时间同步前置](./faq/01-integration-questions.md)，调 `/api/v1/stop_and_fetch` 之前必须先调一次 `/api/v1/time_sync` 让盒子知道客户端时钟偏移，否则按服务端时钟取窗口会偏。

> **新员工**：「这套东西到底用来干嘛的？跟普通麦克风录音有啥区别？」
> **你**：直接发 [reference/01-product-overview.md](./reference/01-product-overview.md)。重点是"工业现场长时录音 + 按时间戳精准回放 + 上游算法直调"这套组合，不是消费级录音。

> **运维**：「灵敏度改不动了，PUT 返回 'yaml saved but restart failed'。」
> **你**：先看 [scenarios/02-sensitivity-tuning.md §错误处理](./scenarios/02-sensitivity-tuning.md)，这种情况 yaml 已经写进去了，**重新执行 `systemctl restart SmartAudio`** 即可生效；同时检查 `/home/rock/audio/restart_smartaudio.sh` 是否存在且可执行。

---

## 维护说明（给写 skill 的人看）

- 内容跨多文件，事实来源：
  - 主仓代码：`internal/`、`admin/internal/`、`bin/`、`example/config.yml`
  - 设计文档：`docs/PROJECT_OVERVIEW.md`、`docs/API_HARDWARE.md`、`docs/API_SENSITIVITY.md`、`docs/TIME_SYNC.md`、`docs/BUFFER_POOL_README.md`、`docs/STORAGE_RINGBUFFER.md`、`docs/3CHANNEL_SUPPORT.md`
  - 版本基线：`feature/v3.5` HEAD（v3.5.3 / 2026-05）
  - 历史记忆：`memory/project_hardware_constraints.md`、`memory/box_audio_overflow_hardware.md`、`memory/feedback_deployment_style.md`
- 凡涉及"未来路线"用 *计划 / 路线图 / 设想 / 在调研* 等词修饰，不用 *承诺 / 一定 / 必须 / 即将上线* 等词
- 跨文档引用一律相对路径，方便 GitHub / IDE 渲染
- 内容更新时同步更新 `glossary/terms.md`（如有新概念）
- **永远不写**：内网 IP / hostname / 凭据 / 私钥 / 具体 SN / 真实客户名 / 真实产线名。涉及部署示例时用 `<盒子IP>` `<SN>` 占位
- 安全债类话题（默认 JWT secret / 出厂凭据）只在标了 *(内部参考)* 的章节出现，对外文件干净；**不写明文凭据**

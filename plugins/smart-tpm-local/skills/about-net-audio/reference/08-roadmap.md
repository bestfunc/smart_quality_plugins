# 路线图

> 用词约束：**计划 / 设想 / 在调研 / 路线图阶段**。**不**用 "承诺 / 即将 / 一定 / 必须"。
>
> 任何具体时间点必须打 `<TODO: 待商务/产品确认>` 占位，不要硬编。

## v3.5 已交付（截至 2026-05）

> "已交付" = 已 commit + 已构建 + 已发布到 release/，不代表所有现场盒子都升过。

### 主程序（smartaudio）

- **mDNS 唯一 hostname**：用 `RegisterProxy` 让每台盒子以 `smartaudio-<SN>.local.` 出现，解决出厂镜像 hostname 撞库
- **`POST /api/v1/stop_and_fetch`**：算法直调专用，按时间窗从 BufferPool 取 4 通道音频字节流，单/多通道分别用 audio/wav 和 multipart/form-data 返回
- **A/B/C/D 物理标签翻译**：录音 / stop_and_fetch / upload_audio 等接口的入参支持 `channel_labels`；响应同时返回内部声道号和物理标签
- **SIGTERM 处理**：`systemctl stop` 时能发 mDNS goodbye 包，客户端能感知掉线

### admin（smartaudio-admin）

- **`GET/PUT /api/audio/sensitivities`**：批量灵敏度更新，支持 `sensitivities_by_label` 物理标签入参
  - v3.5.3 路径：写 yaml + exec `restart_smartaudio.sh`（v3.5.2 的"热更新"路径因 GPIO busy 已回退）
- **`GET/PUT /api/network`**：netplan 静态 IP 在线配置
- **`/api/audio/sensitivities` 和 `/api/network` 改为无鉴权**：产品决策，现场算法/PLC 直接调用便利性优先
- **Web UI 网络管理页面**：前端 Vue3 dist 已 embed 进 binary

### 发布工具

- **现场升级包 + 运维手册**：`release/smartaudio_v3.5.3_20260528/`，含 binary + restart 脚本 + discovery 增量配置 + OPS_MANUAL.md + CHANGELOG + SHA256SUMS
- **跨平台编译保持稳定**：Docker `wenjsen/net_audio_xcompile:linux-arm64` 镜像 + QEMU binfmt

### 文档

- `docs/API_HARDWARE.md` / `docs/API_SENSITIVITY.md` 双轨入参示例同步到位
- 现场运维手册（OPS_MANUAL.md）覆盖快速路径 + 完整路径 + 回滚 + 8 类 FAQ

---

## 路线图阶段（设想中，无时间承诺）

### 设想 1：多盒子集群管理

**问题**：当前需要逐台 SSH 升级、改 IP、调灵敏度。客户上量后这套不可持续。

**正在调研的方向**：
- 集中化的 **fleet manager**（中心服务拉取所有盒子状态、批量下发配置 / 升级）
- mDNS 反向代理或 etcd / Consul 风格的服务发现
- 配置 GitOps 化（盒子拉 git 配置而非 admin 推）

### 设想 2：硬件个体差异的厂内基线测试

**问题**：现场两台同型号盒子 stream 带宽差异 4-5 倍，根因是麦克风采集板个体差异（`Input overflowed` 高频率），软件不可修复。

**正在调研的方向**：
- 出厂前增加**逐台 overflow 基线测试**，挑出采集不良板
- 报告随盒子出厂作为质量记录
- 长期看是麦克风板换供应商 / 改方案的事

### 设想 3：安全债清理

**问题**：当前 admin 默认 JWT secret + 出厂默认账号 / 密码，商业交付前必修。其它细节见内部审计报告。

**计划方向**：
- 出厂烧录环节给每台盒子生成唯一 JWT secret
- 首次登录强制改密
- 增加现场审计日志

时间表 **<TODO: 待产品/安全确认>**。

### 设想 4：算法直调接口扩展

**已交付**：`stop_and_fetch` 按时间窗取音频字节流。

**调研中**：
- **流式版本**：算法订阅"未来 N 秒"音频持续推送（类似 stream/live 但带过滤）
- **多盒子时间对齐取数**：算法说"这个时刻所有盒子的音频都给我"，盒子集群协同
- **更细粒度的元数据**：除了音频字节还附带响度 / dBA / 频谱摘要

### 设想 5：SKU 分层

**问题**：当前所有客户拿同一份功能集合。商业模式上想做 Standard / Pro / OEM 分层，但**尚未确定分层依据**。

**调研中**：
- 按"算法直调用 API 速率"分？按"通道数"分？按"是否含集中管理"分？
- 由商务侧主导，技术侧配合做特性开关

---

## 不在路线图上（明确不做）

为节省内部讨论时间，下列方向**当前不计划做**：

- ❌ **盒子里直接跑算法 / 模型推理**：跟产品哲学冲突（采集与算法解耦）
- ❌ **云端服务 / SaaS 化**：客户数据敏感性 + 工业现场网络限制，云端模式不合适
- ❌ **iOS / Android / 移动端**：本系统是固定部署，不针对移动
- ❌ **多语言 SDK**：HTTP/JSON 已经足够通用，不投入做 Python/Java/Go SDK

未来如果业务方向变化可以重新评估，但当前**资源 + 心智都不投在这上面**。

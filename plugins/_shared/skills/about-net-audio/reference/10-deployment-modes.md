# 部署模式

## 三种部署形态

### 1. 单盒子（POC / 小型试点）

- 1 个盒子 + 1 个算法节点（PLC / 边缘服务器 / 工程师笔记本）
- 网络：直连 / 单交换机 / 同 Wi-Fi
- IP：当前盒子出厂静态 IP `<盒子IP>`（按客户提供网段调整）
- 客户运维门槛：低，记住盒子 IP 即可

### 2. 单产线（4-8 台）

- 多台盒子按产线段位 / 工位密度部署
- 1 个中心算法节点订阅所有盒子
- 网络：同一交换机 / 同 VLAN
- IP：必须每台改成不同静态 IP（出厂全是同一个 `<盒子IP>`，详见 §"出厂镜像克隆问题"）
- mDNS 发现：每台用 SN-唯一 host，客户端可一次 `dns-sd -B` 拿全部
- 客户运维门槛：中，需要 IP 规划 + 升级时滚动操作

### 3. 工厂级（几十台规模）

- 多产线、多车间联合部署
- 多个算法节点 + 可能的中央仪表板
- 网络：跨 VLAN / 跨车间，可能需要 mDNS reflect / 反向代理
- 配置一致性：每台盒子配置应该集中下发（**当前路线图阶段**，参考 [reference/08-roadmap.md](./08-roadmap.md)）
- 客户运维门槛：高，建议配运维工具链 + 监控

---

## 出厂镜像的 4 个"克隆即冲突"陷阱

这是当前架构最重要的工程约束，**任何多盒子讨论先想这条**。

| 字段 | 出厂镜像值 | 冲突表现 | 当前缓解 | 长期方案 |
|---|---|---|---|---|
| 系统 hostname | `rockpis`（所有盒子相同） | mDNS A 记录撞 `rockpis.local.` | 代码层 `RegisterProxy` 显式注册 `smartaudio-<SN>.local.`（[reference/04-key-concepts.md §11](./04-key-concepts.md)） | 出厂烧录改 hostname |
| netplan 静态 IP | `<固定 IP>/24`（所有盒子相同） | 多盒子同时上线全部抢同一 IP | mDNS 发现 + admin API `/api/network` 改 IP | 出厂逐台烧不同 IP / 切 DHCP |
| machine-id | 同一份克隆 | DHCP DUID / 某些 systemd 机制撞 | 当前不致命（mDNS 用 SN、IP 静态写死） | 出厂 `rm /etc/machine-id && systemd-machine-id-setup` |
| SmartAudio SN | 同一份克隆（实测两台 MAC 不同的盒子 SN 都是 `<同一 SN>`） | mDNS 唯一 host 会失效（撞同样 SN-host） | 当前**没有缓解**，靠 SN 区分会出错 | 出厂逐台烧唯一 SN（必修） |

**只有 MAC 是真随机的**（每台不同），是目前唯一可靠的硬件区分标识。但客户端拿不到 MAC（要 ARP 扫描，业务层不现实）。

> 这意味着：**靠 SN 区分设备的逻辑在 SN 撞库时会失效**。任何依赖 SN 的接口（含 mDNS 唯一 host）出厂前必须确认 SN 已重置。

## 静态 IP 策略（不切 DHCP）

**产品决策**：不切 DHCP，盒子全程用静态 IP。

为什么不切 DHCP：
- 工业现场 DHCP 池规划不确定，IP 漂移会让算法端的硬编码地址失效
- 多盒子部署希望 IP "可预测"（按工位编号映射 IP）
- 客户的工业网络很多没有 DHCP 服务器或者不允许 DHCP 客户端

应对静态 IP 撞车的工具：
- **mDNS 发现**：每台 SN-唯一 host，客户端先靠 mDNS 找到盒子的当前 IP
- **`PUT /api/network`**：admin 接口直接改 netplan（不重启 systemd-networkd 也能生效）
- **现场顺序通电策略**：换盒子时先断电旧盒子、上电新盒子、改 IP、再恢复旧盒子

详细 IP 改造流程：[scenarios/01-field-deployment.md](../scenarios/01-field-deployment.md)。

## 无 RTC 硬件时钟

Rock Pi S 主板**没有 RTC**。靠：
- `fake-hwclock` 在关机时存盘当前时间
- 开机时从存盘文件恢复

**断电硬关机会丢时间**（fake-hwclock 没机会存）。实测某台跑到 2024-08-01，慢 22 个月。

恢复手段：
- `ntpdate ntp.aliyun.com`（如果盒子可达外网）或客户内网 NTP
- 之后 `fake-hwclock save` 持久化
- 出厂应配置开机自动 NTP 同步

时钟错乱会导致：
- 录音文件名混乱（带时间戳）
- token 校验失败（如果走 JWT）
- `time_sync` 偏移计算错（盒子时钟差太远，客户端时间戳会被认为超出 BufferPool 窗口）

## 安全模型 *(内部参考，不对外发)*

当前安全设计的已知债务：

| 风险 | 当前状态 | 缓解 |
|---|---|---|
| admin 出厂默认账号 / 密码 | 出厂未改 | 客户首次部署应改密；产品计划在出厂烧录环节强制 |
| 默认 JWT secret（`admin-config.yml` 内固定字符串） | 多客户共享同一份 secret | 出厂烧录环节生成唯一 secret |
| `/api/audio/sensitivities` 和 `/api/network` 无鉴权 | 设计决策 | 假设设备只暴露在产线内网；外网部署绝对不行 |
| 主程序 8090 部分接口走 JWT，部分不走 | 历史接口不统一 | 暂不动，新增接口按当前产品决策定 |
| SSH root 出厂默认密码 | 必修 | 客户首次部署改密 + 推免密公钥 |
| `private_key.pem` / `public_key.pem` 在仓库根 | 测试用，生产应分离 | 出厂前替换 + .gitignore（已加） |

**对外口径**：本系统适合**产线内网封闭部署**。绝对不要把 8090 / 8091 暴露到客户公网。

## 升级模式

### 增量升级（推荐）

只换 binary，不动 yaml：
- 备份原 binary + 配置（**绝对不要打包 recordings.narb**）
- scp 新 binary 到盒子 / tmp
- `systemctl stop` → `cp` → `chmod +x` → `systemctl start`
- 校验 is-active + 端口监听 + mDNS 广播

完整流程：[scenarios/01-field-deployment.md](../scenarios/01-field-deployment.md) §滚动升级。

### 增量 + 配置追加

新版本需要新 yaml 字段时（如 v3.5 引入 `discovery:` 段），**追加而非覆盖**：

```bash
if ! grep -q "^discovery:" config.yml; then
  cp config.yml config.yml.pre-vX.Y-<TS>
  cat discovery_increment.yml >> config.yml
fi
```

### 出厂烧录

镜像层一次性解决所有"克隆冲突"：唯一 hostname、唯一 SN、不同静态 IP、唯一 machine-id、唯一 JWT secret、唯一 admin 密码。当前**部分人工**，路线图上**计划自动化**。

## 部署最佳实践 *(内部参考)*

按"过去 12 个月踩过的坑"提炼：

1. **任何升级 / 配置变更前先校时钟**：盒子时间错的话日志时间线和实际操作时间线对不上，事后排故灾难
2. **备份只打 binary + config + service unit，绝对不打 recordings.narb / logs / temp**
3. **`*.pre-<version>-<timestamp>` 命名习惯**：每次替换 binary 把旧的改名留底，回滚一行 `mv` 就行
4. **公钥重推不可省**：换硬件后系统盘公钥可能丢，初次连接用密码 + ssh-copy-id 推回去
5. **改完 IP 先 ping 通再做下一步**：netplan apply 失败会让盒子断网，要靠物理接触恢复
6. **多盒子部署不要同时上电**：撞 IP 之后 ARP 表混乱

# 场景：现场故障排查

8 类高频症状，按"客户在投诉什么"排序。

---

## 1. "盒子连不上"

### 诊断步骤

```bash
# A. 网络层
ping <盒子IP>
# 通 → 继续 B
# 不通 → 网络/电源/网线问题

# B. SSH 22 端口
nc -z <盒子IP> 22
# 通 → ssh
# 不通 → 盒子卡死，需要现场重启电源

# C. 8090 / 8091
curl -s -o /dev/null -w "8090: %{http_code}\n" --connect-timeout 5 http://<盒子IP>:8090/api/v1/time_sync -X POST -d '{"client_time":0}'
curl -s -o /dev/null -w "8091: %{http_code}\n" --connect-timeout 5 http://<盒子IP>:8091/api/auth/login -X POST -d 'foo'
```

| 8090 | 8091 | 解读 |
|---|---|---|
| 200 | 200/400 | 都正常 |
| 000 | 401/200 | 主程序挂了，admin 在跑 → `journalctl -u SmartAudio` |
| 200 | 000 | admin 挂了 → `journalctl -u SmartAudioAdmin` |
| 000 | 000 | 两个都挂或网络层断 |

### 如果 SSH 也连不上但 ping 通

可能是 sshd 卡或者**这台不是原来的盒子**（IP 撞库新机器接进来）。

```bash
# 看 host key 有没有变化
ssh-keyscan -T 5 <盒子IP> 2>&1 | head -3
```

如果显示的 OpenSSH 版本跟原来不一样（比如原来是 Debian 10 配的 8.4，现在是 9.x），**多半是另一台机器**。

修复：
```bash
ssh-keygen -R <盒子IP>   # 清旧 host key
ssh root@<盒子IP>         # 重新连，输入 yes 接受新 key
```

---

## 2. "两台盒子带宽差异巨大 / 一台音频卡顿"

**根因 99% 是麦克风采集硬件个体差异**。

### 一键判断

```bash
ssh root@<盒子IP> 'journalctl -u SmartAudio --since "5 min ago" | grep -c "Input overflowed"'
```

- < 10 次/分钟：正常
- 10-100：边缘
- \> 100：**硬件个体问题**，软件不可修复

### 验证（如果客户不信是硬件问题）

A. **校时钟 + 重启 SmartAudio**，确认 overflow 频率不变（排除时钟波及音频定时）

B. **对比 dmesg 音频驱动**：两台 ALSA 卡 / I2S 驱动相同 → 排除驱动问题

C. **对比配置**：`diff config.yml1 config.yml2` → 应完全一致

D. **硬件对调测试**：把好盒子的麦克风板和坏盒子的机身互换，看 overflow 跟着哪一边走：
- 跟板走 → 换板
- 跟机身走 → 主板 I2S 问题（罕见）

详细历史：agent 记忆里的 `box_audio_overflow_hardware.md`。

---

## 3. "时钟错乱 / 录音文件名乱"

Rock Pi S **无 RTC**，断电会丢时间。

### 现象
- `date` 显示远古时间（2024-08-xx 等）
- 录音文件名按错时间命名
- `time_sync` 算出的偏移巨大

### 修
```bash
ssh root@<盒子IP> bash <<'EOF'
  ntpdate ntp.aliyun.com    # 或客户内网 NTP
  fake-hwclock save
  date
EOF
```

**`fake-hwclock save` 不能省**，否则下次断电再丢。

### 预防
- 开机自动 NTP（systemd-timesyncd 或 chrony）
- 配置可达 NTP 服务器
- UPS（如果客户接受成本）

---

## 4. "灵敏度 PUT 返回 'yaml saved but restart failed'"

含义：yaml 已经改完了，但 SmartAudio 没起来。

### 修
```bash
ssh root@<盒子IP> 'journalctl -u SmartAudio -n 50 --no-pager'
# 找启动失败原因，常见：
# - portaudio 设备占用 → 等 10 秒重试
# - config.yml 语法错误 → admin yaml 写入有问题（罕见）
# - restart_smartaudio.sh 不存在或不可执行 → 拷过去 chmod +x
```

修好后 `systemctl restart SmartAudio` 即可，yaml 是权威值，按新灵敏度生效。

### 常见根因

**`restart_smartaudio.sh` 缺失**：
```bash
ssh root@<盒子IP> 'ls -l /home/rock/audio/restart_smartaudio.sh'
# 如果不存在，从最新 release 包拷
scp release/.../scripts/restart_smartaudio.sh root@<盒子IP>:/home/rock/audio/
ssh root@<盒子IP> 'chmod +x /home/rock/audio/restart_smartaudio.sh'
```

详见 [scenarios/02-sensitivity-tuning.md §错误处理](./02-sensitivity-tuning.md)。

---

## 5. "mDNS 找不到盒子"

### 诊断
```bash
# 在客户端机器
dns-sd -B _smartaudio._tcp    # macOS
avahi-browse -rt _smartaudio._tcp   # Linux

# 看不到任何 SmartAudio-* 实例
```

### 排查
**A. 盒子端 discovery 配置开了吗？**
```bash
ssh root@<盒子IP> 'grep -A2 "^discovery:" /home/rock/audio/config.yml'
# 期望：discovery: \n  enabled: true \n  api_port: 8090
```
如果是 `enabled: false`，改成 `true` 重启 SmartAudio：
```bash
ssh root@<盒子IP> 'sed -i "s/^\(\s*\)enabled: false/\1enabled: true/" /home/rock/audio/config.yml && systemctl restart SmartAudio'
```

⚠️ 注意：**只有 `discovery` 段用 `enabled`（带 d）**，其它 section 是 `enable`。sed 时确认匹配的是 discovery 段下面那行。

**B. 启动日志有没有 mDNS**
```bash
ssh root@<盒子IP> 'journalctl -u SmartAudio --since "5 min ago" | grep -i "mdns\|discovery"'
# 期望：mDNS discovery 已启动: _smartaudio._tcp.local. port=8090
```

**C. 路由 / 网络限制**
mDNS 走 UDP/5353 多播。如果路由器禁多播，跨网段就发现不了。同网段没问题就不用管。

**D. 客户端 mDNS 缓存**
```bash
# macOS
sudo killall -HUP mDNSResponder
# Linux
sudo systemctl restart avahi-daemon
```

---

## 6. "改完静态 IP 盒子失联"

`PUT /api/network` 调完会等 1.5 秒再 `netplan apply`，**应用瞬间所有 TCP 连接会断**。

### 正常流程
1. 客户端 PUT 新 IP
2. 收到 200 响应（盒子还是旧 IP）
3. 等 ~2 秒
4. 用 mDNS 重新发现 → 拿到新 IP
5. 继续工作

### 失败场景

**A. 新 IP 不在子网内 / 网关不可达**：盒子彻底断网，**必须物理接触**。预防是 PUT 之前充分校验。

**B. netplan 配置写坏了**：现场用串口连主板（如果硬件有串口）或拆 SD 卡改 `/etc/netplan/01-ethernet.yaml`。

### 建议
**任何远程改 IP 之前**，先建立"如果失联怎么恢复"的回退计划。

---

## 7. "升级失败 / 服务起不来"

### 诊断
```bash
ssh root@<盒子IP> '
  systemctl status SmartAudio SmartAudioAdmin --no-pager -n 30
'
```

### 立即回滚
见 [scenarios/01-field-deployment.md §回滚](./01-field-deployment.md)。

### 不直接回滚的常见修复

**A. binary 没可执行权限**
```bash
ssh root@<盒子IP> 'chmod +x /home/rock/audio/smartaudio /home/rock/audio/smartaudio-admin'
```

**B. config.yml 缺新版本要求的字段**
v3.5 要求有 `discovery:` 段，老盒子可能没追加。追加后重启。

**C. portaudio 占用**
```bash
ssh root@<盒子IP> 'systemctl reset-failed SmartAudio; sleep 5; systemctl start SmartAudio'
```

---

## 8. "GPIO busy / 灵敏度脚本失败"

```
echo: write error: Device or resource busy
```

### 根因
`SET_TPL0501_FUNC-V1.4.sh` 的 GPIO 初始化非幂等：第一次 `echo X > /sys/class/gpio/export` OK，第二次 EBUSY。

### 不会再触发的场景（v3.5.3+）
v3.5.3 admin 灵敏度更新走 "写 yaml + restart SmartAudio" 路径，主程序在启动时一次性 export GPIO，**不会重复触发**。

### 如果手动跑脚本遇到了
```bash
# 强制 unexport 所有相关 GPIO 再跑
ssh root@<盒子IP> '
  for gpio in <相关 GPIO 编号>; do
    echo $gpio > /sys/class/gpio/unexport 2>/dev/null || true
  done
  bash /home/rock/audio/TPL0501-V1.4.sh ...
'
```

具体编号在 `SET_TPL0501_FUNC-V1.4.sh` 里。

---

## 通用排故节奏 *(给运维同事的建议)*

1. **先看现象**：客户报告的"连不上"是 ping 不通还是 SSH 不通还是 API 不通？三者完全不同
2. **看日志**：`journalctl -u SmartAudio --since "X min ago"`，错误几乎都在这里
3. **比配置**：怀疑配置问题时，跟一台已知好的盒子 `diff config.yml`
4. **比硬件**：同一批次盒子表现有差异，先怀疑硬件个体（特别是麦克风板）
5. **回滚优先**：升级失败、不知道根因，先回滚保业务，再慢慢排查

## 升级问题日志模板 *(给后续整理用)*

每次现场故障建议留一份：

- 日期 / 盒子标识（SN / 工位号）
- 现象
- 客户端 / 盒子两端日志摘录
- 排查路径
- 根因
- 修复动作
- 预防建议（要不要进运维手册 / 改代码）

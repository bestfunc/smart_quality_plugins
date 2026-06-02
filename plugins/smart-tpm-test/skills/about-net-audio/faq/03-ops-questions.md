# FAQ：现场运维常见问题

## Q1：升级失败了怎么办？

**A**：先**回滚**保业务（参考 [scenarios/01-field-deployment.md §回滚](../scenarios/01-field-deployment.md)），再慢慢排查。

回滚步骤：
```bash
ssh root@<盒子IP> bash <<'EOF'
  systemctl stop SmartAudio SmartAudioAdmin
  cd /home/rock/audio
  cp $(ls -t smartaudio.pre-* | head -1) smartaudio
  cp $(ls -t smartaudio-admin.pre-* | head -1) smartaudio-admin
  chmod +x smartaudio smartaudio-admin
  systemctl start SmartAudio SmartAudioAdmin
EOF
```

升级前留下的 `*.pre-<version>-<TS>` 副本就是干这个用的。

---

## Q2：怎么判断盒子升级了没？

**A**：
```bash
curl -s http://<盒子IP>:8091/api/version 2>&1
# 或登录 admin Web 看版本号
```

带 `v3.5.3` 就是最新。

---

## Q3：怎么改盒子静态 IP？

**A**：用 admin 接口：
```bash
curl -X PUT http://<盒子IP>:8091/api/network \
  -H 'Content-Type: application/json' \
  -d '{
    "interface": "eth0",
    "address": "192.168.x.x/24",
    "gateway": "192.168.x.1",
    "dns": ["114.114.114.114"]
  }'
```

**关键**：调完会等 1.5 秒再 `netplan apply`，**这时连接会断**。客户端要：
1. 立刻断开当前连接
2. 等 2-3 秒
3. 用 mDNS 重新发现新 IP

⚠️ 改之前确认新 IP 合法（在子网内、网关可达），否则盒子会断网失联，必须物理接触恢复。

---

## Q4：盒子时钟不准怎么办？

**A**：
```bash
ssh root@<盒子IP> 'ntpdate ntp.aliyun.com && fake-hwclock save'
# 或客户内网 NTP：ntpdate <内网 NTP IP>
```

**`fake-hwclock save` 不能省**，否则下次断电再丢（Rock Pi S 无 RTC）。

预防：开机自动 NTP。配置 systemd-timesyncd 指向可达 NTP 服务器。

---

## Q5：mDNS 找不到盒子怎么办？

**A**：3 步排查：

1. **盒子端**：
   ```bash
   ssh root@<盒子IP> 'grep -A2 "^discovery:" /home/rock/audio/config.yml'
   ```
   `enabled: true` 才行。如果是 false：
   ```bash
   ssh root@<盒子IP> 'sed -i "s/enabled: false/enabled: true/" /home/rock/audio/config.yml && systemctl restart SmartAudio'
   ```

2. **盒子日志**：
   ```bash
   ssh root@<盒子IP> 'journalctl -u SmartAudio --since "5 min ago" | grep -i mdns'
   ```
   期望：`mDNS discovery 已启动: _smartaudio._tcp.local. port=8090`

3. **客户端 mDNS 缓存**：
   ```bash
   # macOS
   sudo killall -HUP mDNSResponder
   # Linux
   sudo systemctl restart avahi-daemon
   ```

详见 [scenarios/04-troubleshooting.md §5](../scenarios/04-troubleshooting.md)。

---

## Q6：换了一台新盒子但 IP 还是同一个（撞库）

**A**：出厂镜像所有盒子静态 IP 都是同一份。同网段同时上线会抢同一个 IP。

应对：
- **先断电旧盒子**（避免 ARP 表混乱）
- **上电新盒子**
- **改新盒子 IP**（用 mDNS 找到，PUT /api/network）
- **恢复旧盒子上电**

详见 [reference/10-deployment-modes.md](../reference/10-deployment-modes.md)。

---

## Q7：SSH 密码 / 公钥都不行怎么办？

**A**：
1. **看看是不是 host key 变了**：
   ```bash
   ssh-keygen -R <盒子IP>
   ssh root@<盒子IP>   # 输 yes 接受新 key + 密码 rock
   ```

2. **如果之前推过公钥但现在不认了**：盒子可能换硬件（系统盘换了），公钥得重推：
   ```bash
   ssh-copy-id root@<盒子IP>
   ```

3. **密码不对**：用出厂交付清单里的 SSH root 默认密码（**内部参考，不在本文档列明**）。如果客户改过密码但没记，**只能物理接触盒子用串口 / SD 卡重置**。

---

## Q8：升级后客户反映"录音质量差了"

**A**：先确认是不是 overflow：
```bash
ssh root@<盒子IP> 'journalctl -u SmartAudio --since "10 min ago" | grep -c "Input overflowed"'
```

- 如果 < 10/分钟：不是录音质量问题，可能是灵敏度问题，跟客户对齐"什么叫质量差"
- 如果 > 100/分钟：**麦克风采集硬件个体问题**，软件不可修复，建议换板。详见 [scenarios/04-troubleshooting.md §2](../scenarios/04-troubleshooting.md)

---

## Q9：admin 登录用什么账号？

**A**：出厂默认账号 / 密码见出厂交付清单 *(内部参考，不在本文档列明)*。

⚠️ 客户首次部署应该改密。改法：admin Web UI → 用户设置 → 改密码，或先 `POST /api/auth/login` 拿 JWT，再调改密 API。

具体改密 API 走 admin 的 user 管理路径，参考 `admin/README.md`。

---

## Q10：灵敏度调多少合适？

**A**：

- **不知道现场情况** → 出厂默认 240（中等增益），试录一段看
- **远场（>2m） / 噪声小** → 调高，试 200-255
- **近场（<0.5m） / 噪声大 / 出现削波** → 调低，试 100-150

4 个通道可以独立调（按物理位置 A/B/C/D 不同距离补偿）。

调法：
```bash
curl -X PUT http://<盒子IP>:8091/api/audio/sensitivities \
  -H 'Content-Type: application/json' \
  -d '{"sensitivities_by_label":{"A":180,"B":200,"C":120,"D":150}}'
```

⚠️ 这一调 SmartAudio 会重启，**录音中断约 5 秒**。详见 [scenarios/02-sensitivity-tuning.md](../scenarios/02-sensitivity-tuning.md)。

---

## Q11：盒子的存储 / 录音文件在哪？

**A**：
- 内存缓冲（最近 30 秒）：BufferPool，无文件可见
- 磁盘环形：`/home/rock/audio/recordings.narb`（默认 10GB，覆盖式写入）
- legacy 录音队列：`/home/rock/audio/<日期>/channel<N>/*.wav`（如果开了 storage.dir，已逐步淘汰）

调试 / 取历史音频走 admin Web UI 的"存储管理"页（或 `/api/storage/entries` 接口）。

---

## Q12：怎么看实时日志？

**A**：
```bash
# 主程序
ssh root@<盒子IP> 'journalctl -u SmartAudio -f'
# admin
ssh root@<盒子IP> 'journalctl -u SmartAudioAdmin -f'

# 或文件
ssh root@<盒子IP> 'tail -f /home/rock/audio/smartaudio.log'
```

或者 admin Web UI → "日志管理" → 实时日志流。

---

## Q13：盒子能告诉我自己是哪一台吗？

**A**：

```bash
curl -s http://<盒子IP>:8091/api/service/version
# 或主程序：
ssh root@<盒子IP> '/home/rock/audio/smartaudio version'
```

返回会有 SN、版本号。⚠️ **如果是出厂未重置的盒子，SN 跟其它盒子是一样的**（克隆问题）。靠 MAC 区分更可靠：
```bash
ssh root@<盒子IP> 'ip a | grep ether'
```

---

## Q14：客户问"产线断电恢复后录音连续吗"

**A**：

- 断电期间的音频**丢失**（盒子停了，没在采）
- 重启后 BufferPool 是空的（最近 30 秒得现采）
- RingBuffer (`recordings.narb`) 在磁盘上保留，断电前的历史还在
- 客户端要重新 mDNS 发现 + 重新 time_sync

如果客户要求**断电零丢失**，需要 UPS（不在产品默认范围内，客户自配）。

---

## Q15：升级前要不要通知客户？

**A**：建议通知，因为：
- SmartAudio 重启会让录音中断几秒
- WebSocket stream 会断开（客户端要重连）
- mDNS goodbye + 重新注册（中间有几秒不可发现）

如果是停产维护时间窗操作就不用特别通知。生产时间操作要协调。

---

## Q16：怎么知道一台盒子是不是健康的？

**A**：一个简单巡检脚本：

```bash
#!/bin/bash
IP=$1
echo "=== $IP ==="
echo "时间: $(ssh root@$IP date)"
echo "服务: $(ssh root@$IP 'systemctl is-active SmartAudio SmartAudioAdmin' | tr '\n' ' ')"
echo "端口: $(ssh root@$IP "ss -lntp 2>/dev/null | grep -cE ':8090|:8091'") / 2"
echo "overflow/5min: $(ssh root@$IP 'journalctl -u SmartAudio --since "5 min ago" | grep -c Input overflowed')"
echo "uptime: $(ssh root@$IP uptime -p)"
```

健康标准：
- 时间在当前年份
- 两个服务都 active
- 端口都 LISTEN
- overflow < 50/5min
- uptime 合理（不是刚重启）

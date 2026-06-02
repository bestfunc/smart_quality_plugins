# 场景：现场部署 / 升级

## 角色

现场运维工程师 / AI 辅助部署

## 适用前提

- 盒子已通电、有静态 IP、可 SSH（出厂默认密码或公钥）
- 你电脑上有交付包（`release/smartaudio_v3.5.x_<日期>.tar.gz`）

## 通用前置步骤（**所有场景都先做**）

### 1. 校时钟

```bash
ssh root@<盒子IP> bash <<'EOF'
  date
  ntpdate ntp.aliyun.com    # 或客户内网 NTP
  fake-hwclock save
  date
EOF
```

期望：年份显示 2026（或当前年份）。如果跑到 2024-08 这种远古时间，**必须先校再做任何操作**，否则录音文件名、token 校验、日志时间线全乱。

### 2. 最小备份（绝对不打 recordings.narb）

```bash
ssh root@<盒子IP> bash <<'EOF'
  TS=$(date +%Y%m%d_%H%M%S)
  tar -czf /root/smartaudio_min_backup_${TS}.tar.gz -C / \
    home/rock/audio/smartaudio \
    home/rock/audio/smartaudio-admin \
    home/rock/audio/config.yml \
    home/rock/audio/admin-config.yml \
    etc/systemd/system/SmartAudio.service \
    etc/systemd/system/SmartAudioAdmin.service \
    home/rock/audio/restart_smartaudio.sh 2>/dev/null || true
  ls -lh /root/smartaudio_min_backup_${TS}.tar.gz
EOF
```

期望大小：30~50MB（两个 binary + 几个文本文件）。如果出来几百 MB → 误打了 `recordings.narb`，立刻删重打。

### 3. 上传交付包

```bash
scp -r smartaudio_v3.5.x_<日期> root@<盒子IP>:/tmp/smartaudio_release
```

---

## 场景 A：首次部署（盒子还没装过 SmartAudio）

### A.1 检查是不是干净的 Rock Pi S

```bash
ssh root@<盒子IP> 'cat /etc/os-release | head -3; ls /home/rock/audio 2>&1 | head -5'
```

期望：`Debian GNU/Linux 10 (buster)` + `/home/rock/audio` 不存在或为空。

### A.2 创建部署目录 + 拷贝文件

```bash
ssh root@<盒子IP> bash <<'EOF'
  mkdir -p /home/rock/audio
  cp /tmp/smartaudio_release/bin/* /home/rock/audio/
  cp /tmp/smartaudio_release/scripts/restart_smartaudio.sh /home/rock/audio/
  chmod +x /home/rock/audio/smartaudio /home/rock/audio/smartaudio-admin \
           /home/rock/audio/restart_smartaudio.sh
EOF
```

### A.3 写 config.yml（首次必须）

参考 `example/config.yml` + 交付包里的 `discovery_increment.yml`，写一份完整的配置：

```yaml
audio:
  sample_rate: 44100
  bits_per_sample: 16
  channels: 4
  split_channels: 4
  device_index: 2          # 跑 `smartaudio devices` 确认是 dummy-card hw:1,0 对应的 index
  sensitivities: [240, 240, 240, 240]
buffer_pool:
  enable: true
  duration: 30             # 保留最近 30 秒
  channels: 4
storage:
  enable: true
  file: /home/rock/audio/recordings.narb
  max_size: 10737418240    # 10GB
upload:
  enable: false            # 走 stop_and_fetch 路径时建议关
  endpoint: ""
  interval: 1000
discovery:
  enabled: true            # 多盒子部署必须 true
  api_port: 8090
realtime:
  enable: false
remote_config:
  enable: false
logger:
  level: INFO
  file: /home/rock/audio/smartaudio.log
persistence:
  config_file: .smartaudio
system:
  sensitivity_script: /home/rock/audio/TPL0501-V1.4.sh
```

### A.4 装 systemd unit

```bash
ssh root@<盒子IP> bash <<'EOF'
  cat >/etc/systemd/system/SmartAudio.service <<'UNIT'
[Unit]
Description=SmartAudio
After=network.target
[Service]
ExecStart=/home/rock/audio/smartaudio run --config /home/rock/audio/config.yml
WorkingDirectory=/home/rock/audio
Restart=on-failure
[Install]
WantedBy=multi-user.target
UNIT

  cat >/etc/systemd/system/SmartAudioAdmin.service <<'UNIT'
[Unit]
Description=SmartAudio Admin
After=network.target
[Service]
ExecStart=/home/rock/audio/smartaudio-admin --config /home/rock/audio/admin-config.yml
WorkingDirectory=/home/rock/audio
Restart=on-failure
[Install]
WantedBy=multi-user.target
UNIT

  systemctl daemon-reload
  systemctl enable SmartAudio SmartAudioAdmin
  systemctl start SmartAudio SmartAudioAdmin
  sleep 5
  systemctl is-active SmartAudio SmartAudioAdmin
EOF
```

期望：两个都 `active`。

### A.5 改默认 admin 账号密码 + JWT secret（**强烈建议**）

详见 [reference/10-deployment-modes.md §安全模型](../reference/10-deployment-modes.md)。

---

## 场景 B：滚动升级（盒子已经在跑老版本）

### B.1 判断升级路径

```bash
ssh root@<盒子IP> '
  ls /home/rock/audio/restart_smartaudio.sh 2>&1
  grep -c "^discovery:" /home/rock/audio/config.yml
'
```

判断：

| 现状 | 走 |
|---|---|
| 有 `restart_smartaudio.sh`、`config.yml` 有 `discovery:` 段 | **快速路径**（B.2，只换 binary） |
| 缺一个或两个都缺 | **完整路径**（B.3，补脚本 + 追加 config） |

### B.2 快速路径：只换 binary

```bash
ssh root@<盒子IP> bash <<'EOF'
  set -e
  TS=$(date +%Y%m%d_%H%M%S)
  cd /home/rock/audio

  systemctl stop SmartAudio
  mv smartaudio smartaudio.pre-v3.5.x-$TS
  cp /tmp/smartaudio_release/bin/smartaudio ./smartaudio
  chmod +x smartaudio

  systemctl stop SmartAudioAdmin
  mv smartaudio-admin smartaudio-admin.pre-v3.5.x-$TS
  cp /tmp/smartaudio_release/bin/smartaudio-admin ./smartaudio-admin
  chmod +x smartaudio-admin

  cp /tmp/smartaudio_release/scripts/restart_smartaudio.sh ./restart_smartaudio.sh
  chmod +x restart_smartaudio.sh

  systemctl start SmartAudio SmartAudioAdmin
  sleep 5
  systemctl is-active SmartAudio SmartAudioAdmin
EOF
```

### B.3 完整路径：补脚本 + 追加 config + 换 binary

```bash
ssh root@<盒子IP> bash <<'EOF'
  set -e
  cd /home/rock/audio

  # 1. 追加 discovery 段（若缺）
  if ! grep -q "^discovery:" config.yml; then
    cp config.yml config.yml.pre-v3.5-$(date +%Y%m%d_%H%M%S)
    cat /tmp/smartaudio_release/config/discovery_increment.yml >> config.yml
    echo "[OK] discovery 段已追加"
  fi

  # 2. 部署 restart 脚本
  cp /tmp/smartaudio_release/scripts/restart_smartaudio.sh ./restart_smartaudio.sh
  chmod +x restart_smartaudio.sh

  # 3. 换 binary（同 B.2）
  TS=$(date +%Y%m%d_%H%M%S)
  systemctl stop SmartAudio SmartAudioAdmin
  mv smartaudio smartaudio.pre-v3.5.x-$TS
  mv smartaudio-admin smartaudio-admin.pre-v3.5.x-$TS
  cp /tmp/smartaudio_release/bin/smartaudio /tmp/smartaudio_release/bin/smartaudio-admin .
  chmod +x smartaudio smartaudio-admin
  systemctl start SmartAudio SmartAudioAdmin
  sleep 5
  systemctl is-active SmartAudio SmartAudioAdmin
EOF
```

---

## 场景 C：多盒子滚动升级（产线现场）

按顺序升每一台：

```bash
for ip in <盒子IP-1> <盒子IP-2> <盒子IP-3> ...; do
  echo "=== 升级 $ip ==="
  # 1. 校时钟
  ssh root@$ip "ntpdate ntp.aliyun.com && fake-hwclock save"
  # 2. 备份
  ssh root@$ip "TS=\$(date +%Y%m%d_%H%M%S); tar -czf /root/bk_\$TS.tar.gz -C / home/rock/audio/smartaudio home/rock/audio/smartaudio-admin home/rock/audio/config.yml"
  # 3. 上传 + 升级
  scp -r smartaudio_v3.5.x_<日期> root@$ip:/tmp/smartaudio_release
  ssh root@$ip "bash /tmp/smartaudio_release/upgrade.sh"   # 或手抄上面的 B.2/B.3
  # 4. 验证
  curl -s http://$ip:8091/api/auth/login -X POST -d 'foo' || echo "$ip admin 通"
  echo "=== $ip 完成 ==="
  sleep 5
done
```

⚠️ **不要并行升**：一台失败时人需要立即介入，并行的话状态难追踪。

---

## 验证清单（每台升完都跑一遍）

```bash
ssh root@<盒子IP> 'systemctl is-active SmartAudio SmartAudioAdmin'
# 期望：active / active

ssh root@<盒子IP> "ss -lntp | grep -E ':8090|:8091'"
# 期望：两个端口都 LISTEN

curl -s http://<盒子IP>:8091/api/version 2>&1 | grep -o 'v[0-9.]*'
# 期望：升级后的版本号

# mDNS 验证（在同网段任一 mac/linux）
dns-sd -B _smartaudio._tcp
dns-sd -L SmartAudio-<sn> _smartaudio._tcp local
# 期望 host = smartaudio-<sn>.local.

# stop_and_fetch 烟测（无 token 应返 401，证明路由可达）
curl -i -X POST "http://<盒子IP>:8090/api/v1/stop_and_fetch" \
  -H 'Content-Type: application/json' \
  -d '{"channels":[1],"detect":{"detectStageID":"t","startTime":0,"endTime":0},"return_audio":true}'

# 过 5 分钟看一下 overflow 基线
ssh root@<盒子IP> 'journalctl -u SmartAudio --since "5 min ago" | grep -c "Input overflowed"'
# 经验值：<10/分钟正常，>100 是采集硬件个体问题
```

---

## 回滚

```bash
ssh root@<盒子IP> bash <<'EOF'
  set -e
  cd /home/rock/audio
  systemctl stop SmartAudio SmartAudioAdmin

  if ls smartaudio.pre-v3.5.x-* >/dev/null 2>&1; then
    mv smartaudio smartaudio.failed-$(date +%H%M%S)
    cp "$(ls -t smartaudio.pre-v3.5.x-* | head -1)" smartaudio
    chmod +x smartaudio
  fi
  if ls smartaudio-admin.pre-v3.5.x-* >/dev/null 2>&1; then
    mv smartaudio-admin smartaudio-admin.failed-$(date +%H%M%S)
    cp "$(ls -t smartaudio-admin.pre-v3.5.x-* | head -1)" smartaudio-admin
    chmod +x smartaudio-admin
  fi

  systemctl start SmartAudio SmartAudioAdmin
  sleep 3
  systemctl is-active SmartAudio SmartAudioAdmin
EOF
```

如果 binary 备份也丢了，从 `/root/smartaudio_min_backup_*.tar.gz` 解：

```bash
ssh root@<盒子IP> "tar -xzf /root/smartaudio_min_backup_<TS>.tar.gz -C / && systemctl restart SmartAudio SmartAudioAdmin"
```

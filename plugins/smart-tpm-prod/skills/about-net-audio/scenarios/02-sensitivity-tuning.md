# 场景：灵敏度调校

## 你在干嘛

调 4 个通道（A/B/C/D 物理标签）的麦克风采集灵敏度，让现场算法拿到的音频既不过曝（爆音）也不太弱（信噪比差）。

## 前置知识

- 4 个通道**互相独立**（物理标签 A B C D，内部声道 3 4 1 2）
- 每个通道一个 0-255 的整数，**越大增益越大** —— 远处声音也能录，但噪声底也抬
- 配置以 `config.yml` 的 `audio.sensitivities` 为权威源
- TPL0501 数字电位器是硬件控制端，由 `bin/TPL0501-V1.4.sh` 脚本通过 GPIO sysfs 写入
- **改灵敏度要重启 SmartAudio**，录音会中断约 5 秒

## 调校链路（v3.5.3 当前路径）

```
[admin API: PUT /api/audio/sensitivities]
  → service.SensitivityManager.Apply()
  → 校验 4 个值 ∈ [1, 255]
  → 写 /home/rock/audio/config.yml（audio.sensitivities 段）
  → exec /home/rock/audio/restart_smartaudio.sh
    → systemctl stop SmartAudio
    → sleep 2s（让 portaudio / GPIO 完全释放）
    → systemctl start SmartAudio
      → 主程序读 config.yml
      → exec system.sensitivity_script（即 TPL0501-V1.4.sh）
      → SET_TPL0501_FUNC-V1.4.sh 通过 GPIO 写 TPL0501 → 4 个通道生效
    → 轮询 is-active（最多 10s）
  → 返回响应
```

整个过程约 5 秒。

## 接口

### GET 当前值

```bash
curl http://<盒子IP>:8091/api/audio/sensitivities
```

返回（同时给两种视角）：
```json
{
  "success": true,
  "data": {
    "sensitivities": [240, 240, 240, 240],
    "sensitivities_by_label": { "A": 240, "B": 240, "C": 240, "D": 240 }
  }
}
```

### PUT 新值（两种入参互斥）

```bash
# 按声道顺序
curl -X PUT http://<盒子IP>:8091/api/audio/sensitivities \
  -H 'Content-Type: application/json' \
  -d '{"sensitivities":[120,150,180,200]}'

# 按物理标签（推荐）
curl -X PUT http://<盒子IP>:8091/api/audio/sensitivities \
  -H 'Content-Type: application/json' \
  -d '{"sensitivities_by_label":{"A":180,"B":200,"C":120,"D":150}}'
```

**两种表达同一份配置**（A=ch3, B=ch4, C=ch1, D=ch2）。

## 调校经验

### 起步基线
出厂默认 240 是中等增益。如果不知道现场该多少，**先用 240 试录一段**，看上游算法返回的 dBA / 峰值再调。

### 调高的场景
- 远场录音（麦克风离声源 > 2m）
- 现场环境噪声本来就小
- 上游算法觉得"信号太弱"

### 调低的场景
- 近场（< 0.5m）
- 现场有强背景噪声
- 录音出现削波 / 爆音

### 4 通道不必都一样
- 物理位置不同 → 离声源距离不同 → 同一个值可能差很多
- 不同位置的麦克风可能需要不同灵敏度补偿空间衰减

## 错误处理

### `400` 校验失败
```json
{"success":false,"error":"sensitivities must have exactly 4 values, got 3"}
```
- 长度必须是 4
- 每个值 ∈ [1, 255]
- 两种入参不能同时给

### `500` "yaml saved but restart failed"
```json
{"success":false,"error":"yaml saved but restart failed: exit status 1",
 "script_output":"systemctl stop failed: ..."}
```

含义：**新值已经写进 yaml**，但 SmartAudio 重启失败。

应对：
1. 上盒子看 `journalctl -u SmartAudio -n 50 --no-pager`，看 SmartAudio 为什么起不来
2. 修好启动问题后 `systemctl start SmartAudio`，灵敏度按新值生效（yaml 是权威）

常见原因：
- `restart_smartaudio.sh` 不存在或不可执行（v3.5.x 前版本没这脚本，admin 升级了但脚本没拷过去）
- portaudio 卡住没释放（罕见，再 systemctl reset-failed + start）

### `500` yaml 写入失败
极少见，通常是磁盘满或权限异常。看 `df -h` 和 `ls -l /home/rock/audio/config.yml`。

## 为什么不是"热更新"

v3.5.2 曾尝试过"admin 直接 exec TPL0501-V1.4.sh，不重启 SmartAudio"的热更新路径。

**问题**：`SET_TPL0501_FUNC-V1.4.sh` 里的 GPIO 初始化代码非幂等：
```bash
echo $gpio > /sys/class/gpio/export
```
第一次执行 OK，第二次执行就 `echo: write error: Device or resource busy`（EBUSY，errno 16）。因为脚本没做 "if not exported then export" 的判断，也没做 unexport 清理。

结果：admin 调一次成功，第二次必失败。索性回退到 "写 yaml + restart" 路径（commit 8066c9e）。

**根治方向**（路线图阶段）：改 `SET_TPL0501_FUNC-V1.4.sh` 让 GPIO export 幂等。但优先级低 —— 5 秒中断在现场也能接受。

## 前端 UX 建议

如果做调校 UI：
1. **PUT 后必须断开 stream WebSocket 等 1 秒再重连**（服务端会重启）
2. **拖滑块不要每次都 PUT**：debounce 500ms 或加个"应用"按钮
3. **同时显示 sensitivities + sensitivities_by_label**：让用户在两种视角间切换
4. **失败时显示 `script_output`**：是 `yaml saved but restart failed` 还是校验失败，给用户具体错误

## 相关代码 / 文档

- 主程序灵敏度读取：`internal/audio/recorder.go` 启动时 + `bin/TPL0501-V1.4.sh`
- admin Sensitivity Manager：`admin/internal/service/sensitivity.go`
- 接口 handler：`admin/internal/api/handlers/sensitivity.go`
- 物理标签翻译：`admin/internal/service/channel_label.go`
- 完整 API 文档：`docs/API_SENSITIVITY.md`

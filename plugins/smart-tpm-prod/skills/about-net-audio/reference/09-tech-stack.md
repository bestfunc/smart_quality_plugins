# 技术栈

## 一句话

**Go 1.23 + PortAudio + Vue3 + Element Plus** 跑在 **Debian 10 (buster) ARM64** 上。两个 Go 二进制 + 一份 embedded 前端 + 几个 shell 脚本。

## 主程序（smartaudio）

### 语言 / 运行时
- **Go 1.23**（`go.mod` 顶部声明）
- 编译目标：`GOOS=linux GOARCH=arm64`，**CGO 必需**（PortAudio 是 C 库）
- 部署形态：单个静态链接 binary + portaudio runtime（容器镜像内带）

### 核心依赖

| 依赖 | 用途 |
|---|---|
| `gopkg.in/portaudio.v0` | 音频采集（绑定 ALSA） |
| `github.com/gin-gonic/gin` | HTTP / WebSocket 路由 |
| `github.com/gorilla/websocket` | WebSocket 升级 |
| `github.com/grandcat/zeroconf` | mDNS 注册（Register / RegisterProxy） |
| `gopkg.in/yaml.v2` | 配置加载 |
| `go.uber.org/zap` | 结构化日志 |
| `github.com/kardianos/service` | Linux/Windows systemd 注册 |
| `github.com/spf13/cobra` | CLI 子命令 |

### 内部技术细节

- **采样**：默认 44.1kHz / 16bit / 4 通道，帧块 10ms = 441 samples/ch
- **PortAudio 设备索引**：通过 `smartaudio devices` 拿，目标设备是 `dummy-card hw:1,0`（rk3308 上的 I2S 输入）
- **音频缓冲池**：`BufferPool` 持有最近 N 秒 PCM 切片，按 channel 切分
- **WAV 编码**：`internal/utils/audio.go::CreateWavBytes` 把 PCM + sampleRate + bitDepth 编成完整 WAV（含 RIFF 头）
- **mDNS host 派生**：`smartaudio-<lowercase-sn>.local.`，TXT 含 `sn / model / fw`

### 硬件控制

| 操作 | 实现 |
|---|---|
| 灵敏度（4 通道 × 0-255） | `bin/TPL0501-V1.4.sh` + `bin/SET_TPL0501_FUNC-V1.4.sh`（GPIO sysfs 控 SPI 写电位器） |
| 灵敏度脚本 GPIO 初始化 | `SET_TPL0501_FUNC-V1.4.sh`（注意：**非幂等**，重复 `echo X > /sys/class/gpio/export` 会 EBUSY） |
| 服务重启 | `bin/restart_smartaudio.sh`（systemctl stop → sleep 2s → start → 轮询 is-active） |

## admin（smartaudio-admin）

### 语言 / 运行时
- **Go 1.21**
- 编译：`CGO_ENABLED=0 GOOS=linux GOARCH=arm64`（纯 Go，可直接交叉编译）

### 核心依赖

| 依赖 | 用途 |
|---|---|
| `github.com/gin-gonic/gin` | HTTP 路由 |
| `github.com/gorilla/websocket` | 调试 stream WebSocket |
| `github.com/golang-jwt/jwt` | JWT 鉴权（HS256） |
| `golang.org/x/crypto/bcrypt` | 密码哈希 |
| `gopkg.in/yaml.v3` | 配置 / 备份 yaml |

### 前端

| 依赖 | 用途 |
|---|---|
| **Vue 3** + **Vite** | 框架 + 打包 |
| **Element Plus** | UI 组件 |
| Vue Router 4 | 路由 |
| Pinia | 状态管理 |
| Axios | API 调用 |

打包产物 `web/dist/*` → 拷到 `internal/api/static/` → `go:embed all:static` 进 binary。**最终交付是单个 binary 包含 Web UI**，不需要单独部署前端服务器。

### 鉴权链路

- 登录：`POST /api/auth/login` → 验证 bcrypt → 签发 JWT（HS256，admin-config.yml 里的 secret）
- 出厂默认账号 / 密码（**安全债，出厂必修**；具体凭据见出厂交付清单）
- 默认 JWT secret：`admin-config.yml` 里的固定字符串（**安全债，出厂必修**）
- 中间件：`AuthMiddleware` 校验 Bearer token

### 已知豁免（无鉴权）

- `/api/auth/login`：登录入口
- `/api/audio/sensitivities`：v3.5.3 决策，便利性优先
- `/api/network`：v3.5.3 决策

## 系统底层

### OS / 平台

| 项 | 当前选择 |
|---|---|
| 板子 | Rock Pi S（rk3308 SoC，4×Cortex-A35） |
| OS | Debian 10 (buster) ARM64 |
| init | systemd |
| 网络 | netplan 静态 IP |
| 时钟 | **无 RTC**，靠 fake-hwclock 关机存盘 + 开机恢复，断电会丢时间 |
| 音频驱动 | rk3308-acodec + I2S dummy_codec |

### 部署位置

```
/home/rock/audio/
├── smartaudio                    # 主程序 binary
├── smartaudio-admin              # admin binary
├── config.yml                    # 主程序配置
├── admin-config.yml              # admin 配置
├── recordings.narb               # RingBuffer 文件（默认）
├── TPL0501-V1.4.sh               # 灵敏度脚本
├── SET_TPL0501_FUNC-V1.4.sh      # GPIO 初始化
└── restart_smartaudio.sh         # admin 用来重启主程序的脚本（v3.5.3 新增）

/etc/systemd/system/
├── SmartAudio.service
└── SmartAudioAdmin.service

/etc/netplan/
└── 01-ethernet.yaml              # admin 通过 /api/network 改这个
```

## 开发 / 编译

### 主程序（需要 CGO + portaudio）

```bash
# Docker 交叉编译（推荐）
docker run --rm -v "$PWD":/usr/src/myapp -w /usr/src/myapp \
  wenjsen/net_audio_xcompile:linux-arm64 \
  /bin/sh -c 'go build -o ./build/linux-arm64/smartaudio_linux-arm64'
```

Windows 上首次跑要装 binfmt：
```bash
docker run --privileged --rm tonistiigi/binfmt --install arm64
```

### admin（纯 Go）

```bash
cd admin
# 1. 先打前端
cd web && npm run build && cd ..
# 2. 拿版本号
./gen_version.sh
# 3. 交叉编译
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
  go build -o build/linux-arm64/smartaudio-admin ./cmd/admin
```

### 版本号注入

- 主程序：`gen_version.sh` 写 `internal/version.txt` → `internal/cli.go` 用 `go:embed` 读
- admin：`admin/gen_version.sh` 写 `internal/app/version.txt` → `app.go` 用 `go:embed` 读

格式：`<version>,<build_time>,<short_commit>`

## 取舍说明

| 决定 | 原因 |
|---|---|
| Go 而不是 Rust/C++ | Go 编译产物单 binary、跨平台容易、Web 后台生态成熟、采集端用 PortAudio C 库就够 |
| Vue3 而不是 React | 初始团队偏好 + Element Plus 工业风 UI 适用 |
| Element Plus 而不是 Ant Design Vue | 内部统一 |
| 自研 mDNS host 唯一化（RegisterProxy）而不是改 hostname | 不动出厂镜像就能解决多盒子撞库 |
| 灵敏度走 yaml + restart 而不是热更新 | TPL0501 GPIO 非幂等导致 EBUSY，热更新不稳 |
| Vue 打包 embed 进 binary 而不是 nginx 静态托管 | 单 binary 部署，无需额外组件 |
| 不上 Docker 容器化部署 | Rock Pi S 资源紧（512MB RAM），systemd 直跑足够 |
| 不上 Nginx 反向代理 | 内网部署 + 单台盒子，反代收益小于复杂度 |

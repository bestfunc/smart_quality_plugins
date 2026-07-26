# SmartQuality APP V2 安卓端 / 移动采集端

> 交接模块：产线现场用于录音、频谱观察与检测录入的"客户端"层。
> 一句话结论：**真正的安卓端 App 是 `smart_tpm_app`（Flutter 工程），不是 `SmartQuality-client`。**
> `SmartQuality-client`（仓库名 `smart_quality_app_v2`）是 Electron 桌面采集端，只出 Windows/macOS/Linux 包，**不构建安卓 APK**。两者是两套独立代码库、两条产品线，只是都对接同一类声学盒子硬件。

本文所述"客户端/采集端"，指现场操作工在工位上直接使用、负责**采集（录音）**、**看频谱**、**录入检测记录**的软件。它不是后端服务，也不是网页后台。术语首次出现会就地解释；跨文档名词见[术语表](../glossary/terms.md)。

---

## 一、总览：现场到底装了哪个客户端

现场存在两个形态的采集端，容易混淆，先厘清：

| 维度 | `smart_tpm_app` | `SmartQuality-client` |
|------|-----------------|------------------------|
| 定位 | **安卓端 / 手持采集端**（本文主角） | 桌面采集端（Windows 为主） |
| Git 仓库组 | `SMARTTPM/smart_tpm_app` | `platform/smart_quality_app_v2` |
| 技术栈 | **Flutter + Dart**（原生编译） | **Electron + React + TypeScript** |
| 应用显示名 | `SmartTpm智能声学` | `SmartQuality` |
| 出包平台 | **Android APK** + Windows + macOS | Windows(nsis) + macOS(dmg) + Linux(AppImage)，**无安卓** |
| 典型硬件载体 | 商米（Sunmi）安卓手持终端 / 平板 | 工位 PC / 工控机 |
| 版本（撰写时） | `1.9.6-260313+2` | `1.0.4`（release 目录到 1.0.6） |

判定依据（代码事实）：
- `smart_tpm_app` 根目录有 `android/`、`pubspec.yaml`、`lib/*.dart`，`android/app/src/main/AndroidManifest.xml` 声明 `label="SmartTpm智能声学"`，`android/app/build.gradle.kts` 应用了 Flutter Gradle 插件并依赖商米 SDK —— 这是一个可产出 APK 的原生安卓工程（Flutter：Google 的跨平台原生编译框架；Dart：其语言）。
- `SmartQuality-client` 的 `electron-builder.yml` 里 `win/mac/linux` 三个 target **完全没有 Android**；`package.json` 主脚本是 `electron-builder`，用户手册也写"双击 `SmartQuality.exe` 启动"。它虽然仓库名叫 `smart_quality_app_v2`、内部也被叫"移动套装采集端"，但**这里的"移动"指整套设备可推着走，不代表它是手机 App**。它没有用 Capacitor（把 Web 应用包成安卓的方案），也没有任何安卓原生代码。

> 因此，凡说"安卓端 / 手持端"，代码上都指向 `smart_tpm_app`。下文主讲它，并在需要处对照桌面端。

---

## 二、`smart_tpm_app`（安卓端）工程结构与技术栈

### 2.1 技术栈

| 层 | 选型 | 说明 |
|----|------|------|
| 框架 / 语言 | Flutter / Dart（`sdk >=2.17.6 <4.0.0`） | 单套代码编译到 Android/Windows/macOS |
| 路由 | `go_router` | 声明式页面路由 |
| 状态管理 | `hooks_riverpod` + `riverpod_annotation` + `flutter_hooks` | Riverpod：Flutter 的依赖注入 / 状态容器 |
| 数据模型 | `freezed` + `json_serializable` | 生成不可变模型与 JSON 序列化（`*.freezed.dart`/`*.g.dart` 是生成物，勿手改） |
| 网络 | `dio` + `dio_smart_retry` + `talker_dio_logger` | HTTP 客户端、自动重试、日志 |
| 实时通道 | `web_socket_channel`、`udp` | WebSocket 拉实时音频/电平；UDP 用于设备发现/广播 |
| 音频 | `fftea`(FFT)、`wav`、`audioplayers`、`just_audio`(+windows)、`flutter_soloud` | FFT：快速傅里叶变换，把波形转成频谱 |
| 图表 | `fl_chart` | 波形 / 频谱 / 数据曲线 |
| 本地存储 | `localstorage`、SQLite 实体（`build_runner` 生成，见 README） | 离线设置与缓存 |
| 权限/设备 | `permission_handler`、`device_info_plus`、`package_info_plus`、`system_info2` | 运行时权限与设备信息 |
| 手持终端 | 商米 `com.sunmi:SunmiOpenService:1.8.0`（Android 原生依赖） | 通过 MethodChannel `com.bestfunc.smart_tpm/sunmi` 取设备信息 |
| 桌面窗口 | `window_manager`、`desktop_window`、`restart_app` | 供 Windows/macOS 形态使用 |
| 私有依赖 | `flutter_common`（阿里云 codeup 私有 Git 包） | 拉包需要该私有仓库读权限 |

### 2.2 目录结构（`lib/`）

```
lib/
  main.dart / smart_app.dart / router.dart   # 入口、App 装配、路由
  config/        # 配置模型：api / app / database / logger / user_info（freezed）
  api/           # 后端接口封装（api/index.dart 是接口目录）
    dto/         # 请求/响应模型：product、detect_station、quality_detect_*、sunmi_device_info…
  framework/     # 请求底座：requester.dart / app_request.dart（v1↔v2 路径改写 + 鉴权头）
  sensor/        # 声学盒子对接：smart_audio_client、audio_stream_client、modbus_client、realtime
    server/      # 设备侧本地服务相关
  channel/       # sunmi_method_channel.dart（安卓原生桥）
  visualizer/    # 频谱/波形可视化
  ui/screens/    # 页面：home、product_select、detect_station_select、
                 #   quality_detect、auto_quality_detect、detect_record、run_model、setting…
  entity/ provider/ riverpod/ utils/
```

从这个结构可读出核心业务：**选产品 → 选检测工位 → 进入质量检测（人工/自动）→ 录音 + 看频谱 → 出结果 → 记录历史**。

---

## 三、桌面端 vs 安卓端的关系（据代码）

两者**不是同一套代码的不同打包**，而是**两套并行实现**，共享的是"后端契约 + 声学盒子硬件协议"这两处，而非源码：

- **共享的声学盒子**：两端都连一类 HTTP/WebSocket 声学采集盒（内部代号在桌面端 `.env` 里叫 `SV30`）。安卓端 `sensor/smart_audio_client.dart` 用 `Dio` 连 `http://<盒子IP>:8090`，带 `X-Token` 鉴权；桌面端 `.env.example` 也连同一类盒子的 `:8090` WebSocket 流（24bit@96kHz）。协议一致，代码各写各的。
- **各自的后端出口不同**：
  - 安卓端直连"边缘后端"（`smart_tpm_edge_api_v2` / 老 `edge_api`），走 `/api/v2/appapi/v1/*` 或老 `/api/v1/*`，见下节。
  - 桌面端自带一个**内嵌 Express 服务**（本地 `:3001`）处理业务数据，另把登录鉴权指向远程 `SmartQuality-api`。两条后端链路都和安卓端不同。
- **技术栈完全不同**：Flutter/Dart vs Electron/React，没有共享组件。

一句话给接手人：**别指望改一处两端都生效**。安卓端归 `smart_tpm_app`，桌面端归 `SmartQuality-client`，各自独立发版。

---

## 四、与后端接口对接（安卓端调哪些 API）

安卓端有一套 **v1 / v2 双模式** 机制（`lib/framework/app_request.dart`），可在设置里切换（git 历史："新增 v2 appapi 切换开关，支持自定义 X-Workstation-Id"）：

| 维度 | v1（老 edge_api） | v2（smart_tpm_edge_api_v2 appapi） |
|------|-------------------|-------------------------------------|
| 路径前缀 | `/api/v1/*`、`/api/app/v1/*` | `/api/v2/appapi/v1/*` |
| 鉴权头 | `api-key: <token>` | `X-API-Key: <token>` + `X-Workstation-Id: <工位号>` |
| 切换方式 | `AppRequester.setMode(useV2:false,...)` | `setMode(useV2:true, workstationId:...)` |

v2 模式下由一张**路径改写表**把旧路径映射到 appapi（代码里注明"仅 14 个 appapi 接口，其余保持原样"）。已知映射：

| 客户端调用（旧） | 改写到（appapi v2） | 用途 |
|------------------|----------------------|------|
| `/api/v1/products` | `/api/v2/appapi/v1/products` | 产品/型号列表 |
| `/api/v1/category/voice_kind/list` | `/api/v2/appapi/v1/category/voice_kind/list` | 声音种类（结果分类） |
| `/api/v1/sensor/realtime` | `/api/v2/appapi/v1/sensor/realtime` | 传感器实时数据 |
| `/api/app/v1/brandConfig/public` | `/api/v2/appapi/v1/brandConfig/public` | 品牌/外观配置 |
| `/api/app/v1/detectStation*` | `/api/v2/appapi/v1/detectStationApp*` | 检测工位 |
| `/api/app/v1/qualityDetect*` | `/api/v2/appapi/v1/qualityDetectApp*` | 质量检测主流程 |
| 其余（如 `/api/v1/file/download/single`） | 原样透传 | 文件下载等 |
| `_health` | 健康检查（`api/health.dart`，期望返回 `ok`） | 探活 |

DTO（`lib/api/dto/`）覆盖：`product`、`detect_station(_item)`、`detect_point`、`quality_detect_class/main_info/main_history/stage_info/channel_info/result`、`plc_config`、`device_status`、`smart_device`、`sunmi_device_info` 等，基本对应后端质量检测域模型。

> 桌面端的接口清单不同（本地 Express + 远程鉴权），此处不展开，接手桌面端时看 `SmartQuality-client/src/api/*.api.ts`。

---

## 五、权限与硬件调用（安卓端，据 Manifest 与代码）

`android/app/src/main/AndroidManifest.xml` 声明的权限：

| 权限 | 用途（据代码语义） |
|------|--------------------|
| `RECORD_AUDIO` | 录音采集（核心功能） |
| `INTERNET` / `ACCESS_NETWORK_STATE` / `ACCESS_WIFI_STATE` | 连边缘后端、连声学盒子、局域网发现 |
| `READ_MEDIA_IMAGES`（Android 13+） | 读图片（点位图/素材） |
| `READ/WRITE/MANAGE_EXTERNAL_STORAGE` + `requestLegacyExternalStorage=true` | 落地录音/wav、日志、缓存 |

硬件与外设对接：

| 外设 | 代码位置 | 机制 |
|------|----------|------|
| 声学采集盒（SV30 类） | `lib/sensor/smart_audio_client.dart`、`audio_stream_client.dart`、`realtime.dart` | HTTP(`:8090`,`X-Token`) + WebSocket 拉实时音频/电平 |
| PLC / Modbus | `lib/sensor/modbus_client.dart` | TCP `Socket` 定时轮询 Modbus 报文（如 `01 03 …`），用于自动触发/条码转发 |
| 商米手持终端 | `lib/channel/sunmi_method_channel.dart` | Android MethodChannel `com.bestfunc.smart_tpm/sunmi`，取设备序列号等 |
| 扫码枪 | 检测流程（`docs/检测模式流程说明.md`） | 键盘型扫码枪：扫码即输入条码 + 回车启动 |
| UDP 发现 | `udp` 包 | 局域网设备广播/发现 |

**未见蓝牙/串口原生权限**：Manifest 无 `BLUETOOTH*` 权限，盒子走 Wi-Fi/以太网 HTTP+WS，PLC 走 Modbus-TCP。若后续接串口设备需另加原生实现。

检测运行模式（`docs/检测模式流程说明.md`，业务重点）：

| 运行模式 | 触发方式 |
|----------|----------|
| 人工（`manual`） | 手动点开始/结束 |
| 自动（`auto`） | 服务器/PLC 推检测 ID，App 每 2 秒轮询、自动开始/结束 |
| 自动无结果（`autoNResult`） | 同自动，但完成后不显示结果界面 |
| 定时（`time`） | 输入条码后定时触发/结束 |

配合"条码模式（手动/自动）+ 扫码枪开关 + 条码转发 Modbus 开关"组合出完整流程矩阵。

---

## 六、配置项

安卓端配置由 `lib/config/*.dart`（freezed 模型）承载，运行时从本地设置/磁盘 JSON 反序列化：

| 配置模型 | 关键字段 | 说明 |
|----------|----------|------|
| `ApiConfig` | `apiHost`、`audioApiHost`、`audioHost`、`timeout`(默认30)、`apiKey` | 后端地址与鉴权 key |
| `AppConfig` | `deviceId`、`userInfo`、`logger`、`chartFreshInterval`(默认60)、`appTitle`(默认`SmartTPM`) | App 级设置，含图表刷新间隔 |
| `UserInfo` | `username`、`password` | 登录凭据 |
| `DatabaseConfig` | `dir`、`name` | 本地 SQLite 位置 |
| `LoggerConfig` | `dir`、`level` | 日志目录与级别 |

运行时另有 `LocalSettingEntity`（见 `smart_app.dart`）：`platformApi`（后端地址）、`apiKey`、`useV2AppApi`（v1/v2 切换）、`workstationId`（工位号，注入 `X-Workstation-Id`）。这些通常在 App 内"设置页"（`ui/screens/setting.dart`）配置，**现场部署主要就是填后端地址 + apiKey + 工位号 + 盒子 IP/Token**。

> 对照：桌面端配置在 `.env`/`.env.example`（Express 端口、内嵌 PostgreSQL、SeaweedFS、盒子 WS 地址、远程鉴权基地址等），机制与安卓端不同。

---

## 七、打包与签名

### 7.1 安卓端出包（`smart_tpm_app`）

- 构建命令（标准 Flutter）：先 `flutter pub get`，若改过 freezed/json 模型或 SQLite 实体，跑 `flutter packages pub run build_runner build --delete-conflicting-outputs`（见 README），再 `flutter build apk`（或 `appbundle`）。
- 产物：`build/app/outputs/`（APK/AAB）。
- **签名现状（需注意）**：`android/app/build.gradle.kts` 的 `release` 段仍是 `signingConfig = signingConfigs.getByName("debug")`，且带 `// TODO: Add your own signing config`。也就是**当前 release 用的是 debug 签名**，正式分发前一般会补一套自有 keystore（密钥库）与 `key.properties`；仓库内未提交任何 keystore/口令（符合安全要求）。
- **applicationId 仍是脚手架默认值** `com.example.smart_tpm_app`（`namespace` 同）。正式上架前通常需要改成正式包名，否则与"示例包名"冲突、也不利于渠道识别。

### 7.2 桌面端出包（`SmartQuality-client`，对照）

- 命令：`npm run package`（= `build` + `build:electron` + `electron-builder`）；`package:win` 会先 `bump-version` 再出 Windows 包。原生 FFT 插件需 `build:native`（cmake-js 针对 Electron 运行时编译 `fftw-addon`）。
- `electron-builder.yml` 产物：

| 平台 | target | 备注 |
|------|--------|------|
| Windows | `nsis`(x64) | 命名 `SmartQuality-${version}-setup.exe`；`perMachine` 安装；随包裁剪内嵌 PostgreSQL（~125MB→~43MB） |
| macOS | `dmg`(x64+arm64) | 内嵌各架构 PostgreSQL/SeaweedFS/FFTW |
| Linux | `AppImage`(x64) | — |

- 桌面端把 PostgreSQL、SeaweedFS（对象存储）、FFTW 动态库、`.env` 一并 `extraResources` 进包，`asar:false`，`afterPack` 跑 `verify-runtime-deps.cjs` 校验依赖。**签名**：`electron-builder.yml` 未见 Windows 代码签名证书配置（走无证书 NSIS 安装，用户可能遇 SmartScreen 提示）。

---

## 八、已知缺陷与兼容性问题

安卓端（据代码/注释/git）：
- **release 用 debug 签名 + 默认 `com.example.*` 包名**：正式分发前须处理，否则升级覆盖、渠道识别、系统信任都可能出问题。
- **v1/v2 双模式并存**：路径改写表只覆盖 14 个 appapi 接口，v2 下"未在表中的接口原样透传"。新增接口时若忘了加映射，v2 模式会打到不存在的老路径。切换开关配置错误会导致鉴权头（`api-key` vs `X-API-Key`）与后端不匹配而 401。
- **私有依赖 `flutter_common`（阿里云 codeup）**：无该私有仓库读权限则 `pub get` 失败，新机器/CI 首次构建易卡在这里。`smart_tpm_sdk` 私有包在 `pubspec.yaml` 中被注释掉，说明曾用后弃或未启用。
- **存储权限偏宽**：申请了 `MANAGE_EXTERNAL_STORAGE` + `requestLegacyExternalStorage`，在高版本 Android 上架合规审查可能被质疑，且不同 ROM 行为不一致。
- **Modbus 轮询为固定 1 秒 `Timer`**：现场 PLC 慢或断连时的健壮性依赖 `Socket` 异常处理，弱网/掉线重连表现需现场验证。

桌面端（对照，据注释）：
- CSP `connect-src` 必须写成 `http://*:*` 才能连非 80 端口的本地/远程服务（`index.html` 注释）；SV30 未配 `token` 时通道电平极低、回放只有底噪（`.env.example` 注释）。
- 无 Windows 代码签名，首次安装可能触发 SmartScreen 拦截。

---

## 九、后续建议（弱措辞）

- 安卓端正式分发前，或可优先补齐：自有 keystore 签名、正式 applicationId、以及 v2 路径改写表的接口覆盖巡检。
- 双模式（v1/v2）长期可考虑收敛到单一 v2，减少"忘加映射/鉴权头错配"这类隐患。
- `MANAGE_EXTERNAL_STORAGE` 可评估是否能改用分区存储（scoped storage）以利上架合规。
- 若两端未来要共享更多业务逻辑，可评估把"检测流程状态机 + 盒子协议"抽成与语言无关的契约文档（而非共享代码，因栈不同）。
- 文档层面，建议明确对外统一称呼，避免"移动套装采集端"同时指桌面端与安卓端造成的持续混淆。

---

## 十、给接手人：从哪读起 + 如何出包

**安卓端（`smart_tpm_app`）读码路线**：
1. `lib/main.dart` → `lib/smart_app.dart`（App 装配、`init()` 里读 `LocalSettingEntity`、设 apiHost/apiKey/v2/workstationId）→ `lib/router.dart`（页面地图）。
2. `lib/framework/app_request.dart` + `requester.dart`（**先搞懂 v1/v2 与鉴权头**，这是最容易踩坑处）。
3. `lib/api/index.dart` + `lib/api/dto/`（后端契约）。
4. `lib/sensor/`（盒子、Modbus、实时通道）与 `lib/channel/sunmi_method_channel.dart`（安卓原生桥）。
5. `docs/检测模式流程说明.md`（业务流程矩阵，务读）。
6. 页面从 `lib/ui/screens/home` 入，跟着 `product_select → detect_station_select → quality_detect / auto_quality_detect → detect_record` 走一遍。

**安卓端出包**：
```
flutter pub get
flutter packages pub run build_runner build --delete-conflicting-outputs   # 改过模型时
flutter build apk            # 或 flutter build appbundle
```
正式包记得先配好 keystore 与正式 applicationId（见 §7.1）。产物在 `build/app/outputs/`。

**桌面端（`SmartQuality-client`）读码路线**：`electron/main.ts`（启动/异常/内嵌服务）→ `electron/services/*`（Express/PostgreSQL/SeaweedFS/updater）→ `src/api/*.api.ts`（业务接口）→ `src/views/*`。出包：`npm run build:native` 一次，随后 `npm run package:win`（或 `package`）。

---

相关文档：[移动套装](./06-mobile-suite.md)、[术语表](../glossary/terms.md)。

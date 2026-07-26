# 微信服务平台（wxdevops-task-admin）

> 交接文档模块 · 系统篇 01
> 对应仓库：`wxdevops-task-admin`（后端主体，内部代号 bestfunc-wechat）+ `bestfunc_wechat_plugin`（Claude Code 插件包）

## 这是什么 / 为什么存在

**微信服务平台**是 SmartQuality 生态里负责"把企业微信群里的对话变成结构化、可检索、可授权外读的数据"的后端子系统。它做三件事：

1. **监听并存档**产线相关企业微信群的聊天消息（文本、图片、视频、语音、文件），把加密的会话内容解密后落库；
2. **AI 分析 + 云效同步**——用大模型识别群里"谁把什么任务派给了谁"，自动到阿里云云效（Codeup/Projex 体系，内部称"云效"）创建工作项（此链路当前处于禁用状态，见 §9）；
3. **只读对外开放**——通过一个自建的 OAuth 2.1 授权服务器 + MCP Server（Model Context Protocol，AI 客户端与工具/数据之间的标准协议），让 AI 助手（如 Claude Code）按用户权限读取"该用户有权限看的群"的聊天记录。

它存在的根因：产线质量问题、客户反馈、任务指派大量发生在企业微信群里，散落且不可检索。这个平台把这些对话**沉淀为组织资产**，既供人在 Web 后台回看，也供 AI 数字员工按授权调用。术语上，本文把"企业微信"简称 **WeCom**（WeChat Work），"会话内容存档"简称**会话存档**（企业微信官方的合规留痕能力）。

一个重要的事实澄清：尽管交接清单里列了"公众号/小程序/服务号/企业微信"，**代码中只实现了企业微信（WeCom）一种形态**——配置项、消息通道、SDK 全部围绕 WeCom 自建应用与会话存档。公众号/小程序/服务号在本仓库中**没有对应实现**，接手时不要误以为存在这些通道。

---

## 一、系统定位与业务流程

整体数据流（源自 `docs/企业微信会话存档对接指南.md`）：

```
企业微信服务器 ──回调通知──▶ 本系统回调接口(/api/callback)
                              │
                              ▼
                   拉取会话存档消息（C SDK / Python 封装）
                              │
                     消息解密(RSA+AES) → 存入 MySQL
                              │
                  ┌───────────┼───────────┐
                  ▼           ▼           ▼
              AI 分析     媒体下载     @提及回调
                  │       (异步Worker)  (推送给指定人)
                  ▼
             同步到云效（创建工作项，当前禁用）
```

同时存在一条**对外只读读取链路**（源自 `_design/05-程序间依赖.md`）：

```
AI 客户端(Claude Code)
  ① OAuth 授权码+PKCE → 后端 /api/oauth/* 拿 JWT(sub=email)
  ② MCP 调用(Streamable HTTP, Bearer JWT)
  ▼
MCP Server(:8090) —— 校验令牌 + 透传 7 个只读工具
  ▼
后端 Flask(:5000) —— /api/mcp-data/*（按 can_view 过滤）
  ▼
MySQL
```

两条链路共用同一套后端与数据库，区别在于：写入侧（存档、AI、云效）是内部自动流程；读取侧（MCP/OAuth）是对外按用户授权开放的只读接口。详见[架构总览](../reference/02-architecture.md)。

---

## 二、配置项（企业微信 / 会话存档 / 云效 / AI / 存储）

配置以环境变量为主，样例见 `backend/.env.example`，加载逻辑在 `backend/config/settings.py`。**再次强调：仅企业微信一种微信形态，无公众号/小程序/服务号配置。**

| 分组 | 变量 | 含义 |
|---|---|---|
| 企业微信基础 | `WECOM_CORP_ID` | 企业 ID |
| | `WECOM_SECRET` | 自建应用 Secret（拉通讯录/群/成员） |
| | `WECOM_AGENT_ID` | 自建应用 AgentId |
| | `WECOM_API_URL` | 企业微信 API 基址 |
| 会话存档 | `WECOM_ARCHIVE_SECRET` | 会话存档专用 Secret |
| | `WECOM_ARCHIVE_PRIVATE_KEY_PATH` | RSA 私钥文件路径（解密消息用） |
| | `WECOM_ARCHIVE_PRIVATE_KEY` | 私钥内容（与上者二选一） |
| 事件回调 | `WECOM_CALLBACK_TOKEN` | 回调签名校验 Token |
| | `WECOM_CALLBACK_ENCODING_AES_KEY` | 回调加密密钥（43 位） |
| 云效 | `YUNXIAO_API_URL` / `YUNXIAO_ORG_ID` / `YUNXIAO_TOKEN` | 云效地址 / 组织 ID / 个人访问令牌 |
| | `YUNXIAO_DEFAULT_PROJECT_ID` / `_ASSIGNEE_ID` / `_WORK_ITEM_TYPE` | 创建工作项的默认项目 / 处理人 / 工作项类型 |
| AI | `AI_DEFAULT_PROVIDER` | 默认模型：`doubao/dify/openai/claude` |
| | `DOUBAO_API_KEY` / `DIFY_API_KEY` / `OPENAI_API_KEY` / `CLAUDE_API_KEY` | 各家密钥 |
| @提及回调 | `MENTION_CALLBACK_TARGET_USER_ID` / `_NAME` | 默认推送目标（被@时通知谁） |
| 对象存储 | `STORAGE_BACKEND` | `minio` 或 `seaweedfs` |
| | `MINIO_ENDPOINT/ACCESS_KEY/SECRET_KEY/BUCKET/...` | MinIO 连接 |
| | `SEAWEEDFS_FILER_URL` | SeaweedFS Filer 地址 |
| 邮件（登录验证码） | `SMTP_HOST/PORT/USER/PASSWORD/SENDER_NAME/USE_SSL` | 发验证码/通知的 SMTP |
| 数据库 | `MYSQL_HOST/PORT/USER/PASSWORD/DATABASE` | MySQL 连接 |
| 安全 | `SECRET_KEY` | Flask 密钥；MCP JWT 用 `SECRET_KEY + "::mcp-oauth"` 派生 |
| | `CORS_ALLOWED_ORIGINS` | 生产环境允许的前端来源（逗号分隔） |

具体密钥值一律不落文档——生产值放在 `.env.production` / `.env.staging`，通过 docker-compose 的 `env_file` 注入。术语见[术语表](../glossary/terms.md)。

---

## 三、消息与事件处理逻辑（会话存档、回调）

### 会话存档（`services/wecom_archive_service.py` + `wecom_finance_sdk.py`）

- 企业微信提供 **会话内容存档 SDK**（原生 C SDK，本项目用 Python 封装 `wecom_finance_sdk.py` 通过 ctypes 调用）。
- 拉取到的消息是**加密**的：先用 RSA 私钥解出随机对称密钥，再用它 AES 解密消息体。解密后的结构化消息写入 `chat_messages`，媒体类消息在 `chat_attachments` 登记后交由异步 Worker 下载。
- 存档拉取有游标状态表 `archive_sync_state`（记录 seq 进度），支持增量同步、按群重新同步（`/api/wecom/archive/sync/groups`）与类型重判（`resync-type`）。
- 持久化的大头在 `services/archive_persistence_service.py`（约 40KB，是全仓最重的服务文件）。

### 事件回调（`api/callback.py` + `services/wecom_callback_service.py`）

- 路径 `GET/POST /api/callback/wxwork/msgaudit`：GET 用于企业微信回调 URL 的**签名验证握手**（echostr 校验），POST 接收"有新存档消息"的事件通知，触发拉取。
- 回调走企业微信标准的 `Token + EncodingAESKey` 签名与消息加解密。
- CSRF 中间件对 `/api/callback` 路径**豁免**（外部系统调用，无浏览器 cookie 上下文）。

### @提及回调（`api/mention_callback.py` + `services/mention_callback_service.py`）

- 当群消息里 @ 到特定人时，按 `mention_callback_config` 里配置的规则，用 HMAC 签名把消息推送到外部 Webhook（可配多条规则、有 `mention_callback_logs` 留痕、支持测试与示例接口）。这是"被 @ 即通知/转派"的通用扩展点。

### AI 分析（`services/ai_service.py` + `ai_manager.py`）

- 多模型路由：豆包 / Dify / OpenAI / Claude，由 `AI_DEFAULT_PROVIDER` 选默认，`ai_manager.py` 负责按可用性路由。
- 职责是从群对话里识别"任务指派"意图并抽取任务信息，结果落 `task_assignments`，再交由云效服务创建工作项。

---

## 四、用户与权限模型（微信用户 ↔ 云效用户，邮箱即身份）

这里有**两套并存**的身份体系，接手时最容易混淆，务必分清：

**A. 微信用户 ↔ 云效用户映射（业务数据层，为任务同步服务）**

- `wechat_users`：从群成员/客户群导入。`wechat_user_id` 前缀区分来源——无前缀=内部员工原始 userid；`wm/wo`=外部联系人；`dfUser`=手动创建的外部人员；`dfInter`=无 external_userid 的内部员工（系统生成 16 位随机）。唯一性比对顺序：`wechat_user_id → external_userid → unionid → 昵称兜底`（`docs/用户管理逻辑说明.md`）。
- `yunxiao_users` / `yunxiao_projects`：云效侧用户与项目。
- 绑定关系：微信用户可**绑定/解绑**一个云效用户（`POST /api/wechat-users/:id/bind|unbind`），AI 识别出"张三把任务派给李四"后，据此把云效工作项的处理人指向正确的云效账号。

**B. 平台登录与 MCP 授权（账户层，邮箱即唯一身份）**

- **邮箱即唯一账户身份，不与微信用户绑定**（`_design/01-功能设计.md`）。登录=邮箱验证码或 SSO，`session.username = email`。
- RBAC 表：`sys_roles` / `sys_role_permissions` / `sys_user_roles`（均按邮箱授权）。
- **MCP 可见群范围**由该邮箱的权限决定，规则是"并集 + 密级过滤"（`services/group_service.py`）：

  > 可见群 = 本人管理的群 ∪ { 内外部授权放开的整类群中 access_level ≤ 账户密级上限 }

  叠加规则：
  1. 全局群管理 `groups_manage` → 可见/可管全部群，不受密级限制；
  2. 群管理员（按邮箱指派，表 `wechat_group_admins`）→ 其负责的群恒可见、可维护备注/项目/成员备注；
  3. 内外部授权 `WechatGroupScope.scope`（`none/internal/external/all`）→ 在本人群之外**额外放开**整类群（做加法不做减法）；
  4. 密级上限 `WechatGroupScope.clearance`（`public/internal/confidential/secret`）→ **仅**对"授权放开的整类群"做 `access_level ≤ 上限` 过滤，不限本人管理的群。
- 关键分离：`can_view`（MCP 只读可见）与 `can_manage`（Web 后台可编辑/指派）是两套判断，Web 群列表页用 manage，MCP 用 view。

---

## 五、对外接口清单（api/ 层）

后端全部走 Flask 蓝图，前缀由各蓝图注册。挑选核心路由（完整见各 `api/*.py`）：

| 蓝图文件 | 前缀 | 代表接口 | 用途 |
|---|---|---|---|
| `wecom.py` | `/api/wecom` | `status` `groups` `monitor/start\|stop` `token/status\|refresh\|clear` | 连接状态、群列表、监听开关、Token 管理 |
| `wecom.py`（存档段） | `/api/wecom/archive` | `messages` `sync` `sync/groups` `media/download` `decrypt` `status` | 存档消息查询、拉取、媒体下载、解密调试 |
| `wecom.py`（发送段） | `/api/wecom` | `appchat/send` `webhook/send` | 主动发群消息 / 群机器人 Webhook |
| `wechat_user.py` | `/api/wechat-users` | `<id>` `<id>/bind\|unbind` `sync` `refresh-avatar` | 微信用户 CRUD + 绑定云效 |
| `yunxiao.py` | `/api/yunxiao` | `projects` `projects/<id>/members` `tasks` `tasks/<id>/retry` `tasks/batch-retry` | 云效项目/成员/任务同步（蓝图当前未注册，见 §9） |
| `groups.py` | `/api/groups` | `<id>` `<id>/admins` `<id>/members/<uid>` `access-levels` | 群管理：项目/内外部/密级、管理员、成员备注 |
| `mcp_data.py` | `/api/mcp-data` | `me` `groups` `groups/<rid>/messages` `search` `.../attachments` | **MCP 只读数据 API**（Bearer JWT，can_view 隔离） |
| `oauth.py` | `/api/oauth` | `register` `authorize` `authorize/decision` `token` | **OAuth 2.1 授权服务器**（DCR + 授权码 + PKCE + refresh） |
| `auth.py` | `/api/auth` | `send-code` `verify-code` `check` `logout` `sso/login` | 邮箱验证码登录 + SSO 落地 |
| `rbac.py` | `/api/rbac` | `roles` `user-roles` `user-roles/group-scope\|clearance` `my-permissions` | 角色/权限/群授权/密级配置 |
| `mention_callback.py` | `/api/mention-callback` | `configs` `logs` `test` `example` | @提及回调规则管理 |
| `chat_import.py` | `/api/chat-import` | `template` `parse` `confirm` `batches/<id>/rollback` | 离线聊天记录批量导入 + 回滚 |
| `attachment.py` | `/api/attachments` | `<id>/download` `by-sdk-fileid` `<id>/retry` `stats` | 附件下载、按 SDK fileid 取、重试 |
| `callback.py` | `/api/callback` | `wxwork/msgaudit`(GET/POST) | 企业微信事件回调（CSRF 豁免） |
| `sys_config.py` | `/api/sys-config` | `email` `sso` `sso/generate-secret` | 系统配置（SMTP、SSO 密钥） |
| `logs.py` | `/api/logs` | `files` `query` `tail` `stats` | 后台日志查看 |
| `mcp.py` | `/api/mcp` | `tools` `call` `tools/<name>` | 内部 MCP 工具调试端点（CSRF 豁免） |

另有无 `/api` 前缀的 `/.well-known/oauth-authorization-server`（OAuth 元数据发现）与 `/api/health`（健康检查）。

**SSO 单点登录**：本平台既作为 SSO 的**接入方**（`/api/auth/sso/login` 接 BestFunc Kanban 下发的 token），细节与 Level 1/2/3 三级集成方案见 `SSO_INTEGRATION_GUIDE.md`；对外验证接口 `POST /api/sso/verify`（Level 3 回调验证，server-to-server 无需鉴权）。

---

## 六、后端服务架构与 Workers

**分层**（`backend/`）：`api/`（蓝图/路由）→ `services/`（业务）→ `models/models.py`（SQLAlchemy，单文件约 51KB，20 张表）；横切 `middleware/`（请求日志 / CSRF / 登录鉴权）、`config/`、`utils/`、`lib/`（含会话存档 SDK 依赖）。

**应用工厂**（`app.py`）：`create_app()` 注册全部蓝图 → `db.create_all()` 建表 → `_ensure_schema_migrations()` **幂等补列**（给已存在的表加 `project_code/project_name/biz_category/access_level`、`wechat_group_scopes.clearance` 等新列，不用 Alembic）→ `init_default_roles()` 初始化默认角色 → 启动 Worker。

**三个后台 Worker**（都是单例 + `threading`，随 app 启动）：

| Worker | 文件 | 职责 |
|---|---|---|
| 消息监听/处理 | `workers/message_worker.py` | 轮询/监听群消息，入队 `message_queue` 后异步处理（监听线程 + 处理线程双线程模型） |
| 媒体下载 | `workers/media_worker.py` | 从 `chat_attachments` 队列拉媒体消息，下载后上传到 MinIO/SeaweedFS，回写直链 |
| 聊天导入 | `workers/import_worker.py` | 异步处理离线聊天记录批量导入（含 zip 解析、批次回滚） |

> 线程安全与 CSRF/CORS 的收敛改造记录在 `docs/security-thread-safety-changelog.md`（Double Submit Cookie 做 CSRF、生产按 `CORS_ALLOWED_ORIGINS` 收敛来源、修复过多处内存泄漏，见 git 提交 `be7eba5`）。

**MCP Server（协议层，独立进程/容器）**：源码在 `bestfunc_wechat_plugin/src/bestfunc_wechat_mcp/server.py`，同时下沉一份到 `wxdevops-task-admin/mcp-server/`。用 Python **FastMCP（Streamable HTTP）**暴露 7 个只读工具，把调用透传成后端 `/api/mcp-data/*`（携带调用方 Bearer 令牌）；自身作为 OAuth 资源服务器，用 `BearerAuthMiddleware` 调 `/api/mcp-data/me` 校验令牌，并镜像 `/.well-known/oauth-*` 元数据。

**配套插件包 `bestfunc_wechat_plugin`**：这是给 Claude Code 用的 marketplace 插件（Apache-2.0，当前版本见 `plugin.json`），本身**不含服务端代码**。双 MCP 结构：

| MCP | 形态 | 职责 |
|---|---|---|
| `wechat` | 远程 HTTP（`:8090/mcp`） | 读聊天/人员/附件元数据/群信息进上下文 |
| `wechat-file` | 本地 stdio（零依赖 Node，`local-mcp/wechat-file-mcp/`） | 把附件**下载到本机**、图片回传分析、`>=100MB` 跳过；复用连接器 keychain 令牌免二次授权 |

---

## 七、部署形态（docker-compose）

仓库用**基础 + 覆盖**的 compose 分层：

| 文件 | 用途 | 关键点 |
|---|---|---|
| `docker-compose.yml` | 基础定义 | 两服务 `backend`(:5000) + `mcp-server`(:8090)；`wxdevops-network` 桥接网；挂载 `keys`(私钥,只读)、`uploads`、`logs` |
| `docker-compose.dev.yml` | 开发 | 挂载源码目录热更新、`DEBUG=true`、`MYSQL_HOST=host.docker.internal`、`env_file=.env` |
| `docker-compose.staging.yml` | 预发布 | 用 `.env.staging`、外接 `base_docker_default` 外部网络 |
| `docker-compose.prod.yml` | 生产 | 用 `.env.production`、`restart: always`、CPU/内存 limits、不挂源码 |
| `docker-compose.local-db.yml` | 本地 MySQL 8 | 独立起一个 `wxdevops-mysql-dev`（供本地开发/面板管理），不进主编排 |

启动惯例（源自 compose 注释）：
```
开发:   docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
预发布: docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d
生产:   docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
MySQL 未纳入主编排（compose 里被注释掉），生产依赖外部 MySQL；MCP Server 的 `WXDEVOPS_API_BASE` 走容器内服务名 `http://backend:5000`，而 `MCP_RESOURCE_URL`/`AUTH_SERVER_URL` 必须填**公网可直连地址**（客户端和浏览器都要能同时直连 `:8090` 与 `:5000`，否则 OAuth 授权走不通——这是最常见的部署坑）。也有传统 `start.sh`/`start.bat` 直跑（非容器）方式。部署手册见 `docs/部署手册.md`。

---

## 八、前端

`frontend/` 是 React 18 + Ant Design 5 + React Router 6 的 CRA 应用（`react-scripts build`），主要页面：企业微信连接状态、微信用户列表/详情、聊天记录（微信风格存档界面）、任务同步、附件列表、权限管理（`RbacManage.jsx`，含"可看群授权/可见密级上限"两列下拉）。默认开发端口 3000，`/api` 代理到后端 :5000。

---

## 九、已知缺陷 / 未完成 / 后续建议

只列代码与文档能证实的事实：

- **云效同步链路当前禁用**：`app.py` 中 `yunxiao_bp` 蓝图注册被注释（`# 云效模块暂时禁用`）。README 描述的"AI 识别→自动建工作项"主线因此**未在运行**，`yunxiao.py` 接口与 `task_assignments` 表仍在但未挂载。接手若要恢复任务同步，从这里入手。
- **仅企业微信一种形态**：无公众号/小程序/服务号实现，勿按交接标题误判。
- **MCP Server 生产地址待固化**：`plugin.json` 的 `mcpServers.wechat.url` 当前是固定公网 IP（非域名）。`_design/05` 明确建议生产改用域名（如 `wechat-mcp.bestfunc.com`）反代到同一 HTTPS，并把地址回填 plugin.json；当前 IP 属占位/临时。
- **OAuth 双端口暴露风险**：授权要求 `:8090` 与 `:5000` 同时公网可达，生产未收敛到单一 HTTPS 域名前，存在网络策略只放通一个端口导致"卡在未授权"的坑。
- **无正式迁移工具**：schema 演进靠 `_ensure_schema_migrations()` 手写幂等补列，字段一多易漏；后续可**在调研**引入 Alembic。
- **对象存储卷满未处理**：`_design/05` 记"MinIO/SeaweedFS 生产卷满问题独立未处理"。
- **仓库同步状态**：设计文档记"wxdevops 后端改动尚未推 codeup、未上生产"（截至 2026-06-16 该批 MCP/权限改动），接手前先对齐远端仓库与生产实际版本。
- **安全基线**：CSRF 仅生产强制、开发/预发布跳过；`.mcp.json` 里历史上出现过明文 Bearer 令牌（本地调试用），**切勿把真实令牌提交到公开仓库**。

---

## 十、给接手人：从哪读起代码

建议按"先跑通读取链路，再理解写入链路"的顺序：

1. **先读 `_design/` 五份设计文档**（功能/交互/技术/测试/程序间依赖）——这是最新、最贴近当前实现的一手资料，尤其 `03-技术设计.md` 和 `05-程序间依赖.md`。
2. **`backend/app.py`** ——看应用工厂如何装配蓝图、建表补列、拉起三个 Worker，一页掌握全貌。
3. **读取链路**：`api/mcp_data.py` →（鉴权）`services/mcp_auth.py` →（可见性）`services/group_service.py` →（数据）`archive_persistence_service.py`；再看 `api/oauth.py` 理解 OAuth 授权码+PKCE 全流程；MCP 协议层看 `mcp-server/bestfunc_wechat_mcp/server.py`。
4. **写入链路**：`api/callback.py` → `services/wecom_callback_service.py` → `services/wecom_archive_service.py`（+ `wecom_finance_sdk.py` 的 SDK 封装）→ Worker 三件套（`workers/`）。
5. **权限与身份**：`models/models.py` 里搜 `WechatGroupScope` / `WechatGroupAdmin` / `is_groups_global_admin` / `access_level_rank`，配合 `docs/用户管理逻辑说明.md`、`docs/群分类同步逻辑说明.md`。
6. **插件侧**：`bestfunc_wechat_plugin/README.md` + `docs/architecture.md` + `plugins/wechat/skills/*/SKILL.md`，理解 AI 客户端如何安装、授权、读附件。

参见：[架构总览](../reference/02-architecture.md) · [术语表](../glossary/terms.md) · [部署与运维](../reference/03-deployment.md)

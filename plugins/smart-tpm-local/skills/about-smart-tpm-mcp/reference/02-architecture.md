# 02 — 架构

## 定义

本仓库是一个 **Claude Code marketplace**（不是单插件），物理布局上一份"事实源"（`_shared/skills/`）配三个变体目录（每个变体目录的 `skills/` 是 `_shared/skills/` 的物理拷贝）。运行时由 Claude Code 客户端加载某个变体的 `plugin.json`，得到该变体的 MCP server URL + 全套 skill。

## 仓库物理布局

```
SmartTPM_Plugins/
├── .claude-plugin/
│   └── marketplace.json              ← 声明 3 个变体（路径 + 描述）
├── plugins/
│   ├── _shared/
│   │   └── skills/                   ← 9 个业务 skill + 1 本百科 的事实源
│   │       ├── quickstart-smart-tpm-mcp/
│   │       ├── smart-tpm-business-concepts/
│   │       ├── dataset-fields-reference/
│   │       ├── detect-flow-node-reference/
│   │       ├── sample-dataset-extraction/
│   │       ├── locate-test-record-algorithm-chain/
│   │       ├── create-test-task-from-template/
│   │       ├── reproduce-customer-error/
│   │       ├── query-detect-records/
│   │       └── about-smart-tpm-mcp/  ← 本百科
│   ├── smart-tpm-local/
│   │   ├── .claude-plugin/plugin.json  ← URL = localhost:8000
│   │   └── skills/                     ← _shared/skills 的物理拷贝
│   ├── smart-tpm-test/
│   │   ├── .claude-plugin/plugin.json  ← URL = 192.168.2.121:8080
│   │   └── skills/                     ← _shared/skills 的物理拷贝
│   └── smart-tpm-prod/
│       ├── .claude-plugin/plugin.json  ← URL = 192.168.2.175:8080
│       └── skills/                     ← _shared/skills 的物理拷贝
├── README.md
├── CHANGELOG.md
└── generate-product-encyclopedia.md
```

## 为什么物理拷贝，不用 symlink

Windows 上 git symlink 配置麻烦、跨开发机表现不一致，本仓库选择**物理复制**为同步策略：维护者改 `_shared/skills/<x>/` 后，把整个目录 `cp -r` 覆盖到 3 个变体。代价是 3 份冗余、易漂移,所以必须走发版流程一次性同步（详见 [scenarios/02-maintainer-release-workflow.md](../scenarios/02-maintainer-release-workflow.md)）。

## 运行时数据流

```
┌────────────────────────┐                  ┌──────────────────────────┐
│ 用户的 AI 客户端       │                  │ Smart TPM 后端           │
│ (Claude Code / Codex / │                  │ (FastAPI + uvicorn)      │
│  Qwen Code)            │                  │                          │
│                        │                  │  /api/v2/mcp             │
│  + 加载某变体的        │   MCP over HTTP  │  /.well-known/oauth-     │
│    plugin.json         │ ───────────────► │    authorization-server  │
│  + 注册 skill          │                  │                          │
│  + OAuth 2.1 + PKCE    │ ◄─────────────── │  tools (按 scope         │
│    + DCR               │   tool 列表       │    filter)               │
│                        │                  │                          │
│  用户说自然语言        │                  │  ServiceLayer            │
│   ↓                    │                  │   ↓ org/workstation 隔离  │
│  AI 选 skill           │                  │   ↓                      │
│   ↓                    │                  │  MySQL / 工位分库         │
│  AI 调对应 tool        │                  │                          │
└────────────────────────┘                  └──────────────────────────┘
```

**关键设计点**：

- **Transport = HTTP**（不是 stdio）。客户端不需要本机起 server 进程，直接连后端 HTTP endpoint。
- **MCP server 不在本仓库**。本仓库只是声明 URL + 提供 skill 文档。MCP server 本体在 Smart TPM 后端 `api/v2/mcp` 路由下。
- **9 个 skill 不是 MCP server 的一部分**。它们是 Claude Code 端的 markdown 文档 + frontmatter（`allowed-tools` 限定可用工具），由客户端解析。
- **OAuth 走标准 2.1 + PKCE + DCR 流程**：客户端发现 → 注册 → 跳浏览器 → 用户勾 scope → 回调拿 token；token 过期自动 refresh。

## plugin.json 关键字段

每个变体的 `.claude-plugin/plugin.json` 字段：

| 字段 | 说明 |
|---|---|
| `name` | 变体名，全局唯一（如 `smart-tpm-prod`） |
| `version` | 语义化版本，三变体必须保持一致 |
| `description` | 用户在 `/plugin` 列表里看到的一句话说明 |
| `mcpServers.smart-tpm.type` | 固定 `http` |
| `mcpServers.smart-tpm.url` | 该变体对应的后端 MCP endpoint |
| `mcpServers.smart-tpm.httpUrl` | 与 `url` 同值，为兼容 Qwen Code 对 SSE 误识别问题（1.0.2 加入） |

`marketplace.json` 字段：

| 字段 | 说明 |
|---|---|
| `name` | marketplace 名 `smart_quality_plugins` |
| `plugins[].source` | 相对路径 `./plugins/<变体>` |
| `plugins[].description` | 变体一句话说明（用户 `/plugin marketplace add` 后看到） |

## 安装路径

用户侧执行：

```
> /plugin marketplace add bestfunc/smart_quality_plugins
> /plugin install smart-tpm-prod      # 或 smart-tpm-test / smart-tpm-local
```

Claude Code 解析 `marketplace.json` → 进 `plugins/smart-tpm-prod/` → 读 `.claude-plugin/plugin.json` → 注册 MCP server + 加载 `skills/` 下所有 skill → 触发 OAuth → 浏览器同意 → 完成。

## 同 Smart TPM 平台仓库的关系

```
bestfunc/smart_tpm_api  ────►  MCP server 实现 (FastAPI 路由 + tool 注册)
                                       │
                                       ▼
bestfunc/smart_quality_plugins ───►  marketplace + skill 文档 + URL 配置
(本仓库)
```

两个仓库的协作约定：

- 后端新加一个 tool（在 `smart_tpm_api`）→ 本仓库需要更新某个 skill 的 `allowed-tools`，或新增一个 skill 文件 → bump version → 同步到 3 变体
- 后端调整 OAuth scope → 本仓库 `README.md` 的 scope 表 + 相关 skill 描述同步
- 本仓库自己不变后端契约

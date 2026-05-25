# CHANGELOG

## v1.0.5

- 新增第 10 个 skill `about-smart-tpm-mcp` —— 插件 / MCP 集成层的自助百科（user-invocable，按角色分流到 reference / scenarios / faq / glossary）
- 统一各处 scope 描述，去掉硬编码"6 个 scope"等过期数字，补 `mcp:detect_records:*` 一致性
- 同步更新 README / 三变体 `plugin.json` description / `marketplace.json` 至 "10 个 skill / 5 个模块"

## v1.0.4

- 新增第 9 个 skill `query-detect-records`，接入生产 175 的 detect_records 模块（3 个 tool：`_list` / `_get` / `_ping`，工位由 `detect_stations_list` 列出，大文件走 `files_presign_download`）

## v1.0.3

- 新增 `smart-tpm-prod` 生产变体；prod URL 切到内网 IP `http://192.168.2.175:8080`（规避自签证书与客户端兼容性问题，原域名 `https://smartquality.bestfunc.com` 暂不用）

## v1.0.2

- `mcpServers` 三变体均加 `httpUrl` 字段，与 `url` 同值，兼容 Qwen Code 对 SSE transport 的误识别

## v1.0.1

- 重构为 marketplace + 多变体结构：根目录 `.claude-plugin/marketplace.json` 声明 3 个变体（`smart-tpm-local` / `smart-tpm-test` / `smart-tpm-prod`，1.0.1 时只有前两个）
- `_shared/skills/` 物理复制到 3 个变体目录（Windows git symlink 不稳，故采用物理拷贝）
- 删除根 `.mcp.json` 与老 `plugin.json`

## v1.0.0

首发：

- `plugin.json` 声明 MCP server `smart-tpm`（HTTP transport）
- 8 个 Skill（1 Onboarding + 3 Reference + 4 Task），覆盖 datasets / detect_flows / test_tasks / algorithms 4 个模块
- 鉴权走 OAuth 2.1 + PKCE + DCR，配合 Smart TPM Web v1.5.44+

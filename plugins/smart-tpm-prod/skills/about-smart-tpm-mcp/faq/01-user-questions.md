# FAQ — 本地工程师常问

> 装插件 / 用插件中遇到的高频问题。按"装不上 / 授权失败 / tool 不可见 / scope 不够 / 用法疑问"分类。

---

## 装不上

### Q：`/plugin marketplace add bestfunc/smart_quality_plugins` 报 "unsupported source type"

历史上某些 Claude Code 版本对 `marketplace.json` 里 `plugins[].source` 字段格式要求严格。本仓库 1.0.x 之后已统一用相对路径字符串形式，新版客户端兼容。

**应对**：
1. 升级 Claude Code 到最新版
2. 如仍报错，看 marketplace.json 的 `plugins[].source` 字段值（应为 `"./plugins/<变体>"`），如不是，可能是仓库被改坏，找维护者

### Q：装的时候卡在拉取仓库阶段

检查：
- 能否 `git clone https://github.com/bestfunc/smart_quality_plugins` 通
- 公司网络是否拦截 GitHub
- Claude Code 是否配了正确的 git credentials

### Q：装完 `/plugin` 看不到 smart-tpm-prod

```
> /plugin list
[输出不含 smart-tpm-prod]
```

可能 install 没真成功。重装：

```
> /plugin uninstall smart-tpm-prod
> /plugin install smart-tpm-prod
```

---

## 授权失败

### Q：浏览器没自动弹出来

OAuth 跳转依赖 OS 默认浏览器配置。

**应对**：
- 看终端有没有打印一个 URL，手动复制到浏览器打开
- Windows 检查"默认应用 → Web 浏览器"是否设了
- WSL / SSH 环境可能没图形浏览器，需要手动复制 URL 到本机浏览器打开 → 完成授权后会回调到 localhost，需要终端能监听端口

### Q：授权页一直 loading / 报后端连不上

检查：
- 你在公司内网吗？test (192.168.2.121) / prod (192.168.2.175) 都是内网 IP
- `curl http://192.168.2.175:8080/.well-known/oauth-authorization-server` 能否拿到 JSON 响应
- 如果 curl 通但浏览器不通，看浏览器是否走了代理且代理把 192.168.x 排除规则配错

### Q：登录后勾不了 scope（灰掉）

说明你的 Smart TPM 账号还没被授予对应模块的访问权限。找平台管理员开 scope。

### Q：授权完终端没回应

回调端口可能被占。Claude Code 启 OAuth 时会在本机起临时监听端口接收回调，端口冲突或防火墙拦截会导致回调收不到。

**应对**：
- 关掉占端口的程序（如其他 OAuth 客户端、本地 dev server）
- 临时关防火墙试试（确认是这原因后再加白名单）
- 重新 `/plugin install` 重走一次

---

## tool 不可见 / AI 不调用

### Q：`/mcp` 显示 smart-tpm 连上了，但 AI 说 "我没有这个能力"

4 个常见原因：

1. **scope 没勾够**。AI 想调 `datasets_query_samples` 但你授权时没勾 `mcp:datasets:read`。去 Smart TPM Web → 个人设置 → 已连接的应用 → 重新授权勾全。
2. **客户端 skill 缓存**。装完插件后 skill 没立刻被识别。重启 Claude Code（或执行 `/skill reload` 类命令，按客户端版本而定）。
3. **装错变体**。`smart-tpm-local` 连的是本机 8000 端口，你本机没起 docker 时它"连上"也是空。确认是不是该装 `smart-tpm-prod`。
4. **后端那个 tool 真没注册**。罕见，通常发生在 Smart TPM 后端版本与插件 skill 的 `allowed-tools` 不匹配时（如后端老版本、tool 还没上线）。`curl <url> + jsonrpc tools/list` 验证后端实际暴露的 tool 名单。

### Q：AI 调了 tool 但报"权限不足" / "scope 不够"

直接答：去 Web 重新授权勾对应 scope。

scope 对应关系见 [reference/04-key-concepts.md](../reference/04-key-concepts.md) 的 Scope 段。

### Q：AI 调 tool 报 "tool not found"

后端没注册这个 tool 名，但 skill 的 `allowed-tools` 写了。属于本仓库的 bug，找维护者修。临时绕过：换个不依赖该 tool 的 skill。

---

## scope 不够

### Q：我想让 AI 改一下检测流程，为啥不行

故意的。当前 `detect_flows` 模块**不开 `:write` scope**，防止 AI 误改生产流程配置。这是产品决策，详见 [reference/04-key-concepts.md](../reference/04-key-concepts.md) 的 Scope 段。

如果你**真的需要**改检测流程，去 Smart TPM Web 操作。

### Q：写场景就只能新建测试任务？

是。当前约 20 个 tool 里**只有 1 个**写操作：`test_tasks_create_from_template`（基于模板新建测试任务**草稿**）。其他全是只读。

设计意图：插件定位是"读多 + 编排多"，写操作 95% 仍在 Web 完成。

---

## 用法疑问

### Q：我不知道 AI 能做啥，怎么探索

```
> /skill quickstart-smart-tpm-mcp
```

或直接问 AI：

```
> 你都能查 Smart TPM 里哪些东西？
```

AI 会自动匹配 `quickstart-smart-tpm-mcp` skill，列出 5 个模块能力。

### Q：怎么撤销给 AI 的授权

Smart TPM Web → 个人设置 → 已连接的应用 → 找到对应客户端 → 撤销。

撤销后下次 MCP 请求会失败，需要重新 `/plugin install`（或重走授权）。

### Q：换台电脑要重新装吗

是。plugin 安装是 per-machine 的，OAuth token 也存在本机。

### Q：我同时要用本机调试和生产，怎么切

```
> /plugin uninstall smart-tpm-local
> /plugin install smart-tpm-prod
[重新走一次 OAuth]
```

**不要同时装两个变体** —— 三变体的 MCP server key 都是 `smart-tpm`，会覆盖，行为难预测（详见 [reference/03-edition-comparison.md](../reference/03-edition-comparison.md)）。

### Q：AI 调用后我怎么验证它真的干对了

去 Smart TPM Web 对应模块页面看一眼。AI 的所有调用都落 `mcp_audit_log` 表，超管在 Web → 系统设置 → MCP 审计 也能看到完整 trace。

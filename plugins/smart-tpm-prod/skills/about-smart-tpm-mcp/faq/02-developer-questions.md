# FAQ — 维护者常问

> 维护本仓库时的高频问题。按"结构 / 版本 / 同步 / 兼容"分类。

---

## 结构

### Q：为什么不让 Claude Code 直接读 `_shared/skills/`，非要复制到 3 个变体？

因为 Claude Code 的 plugin 加载机制是按 plugin 目录扫的 —— `plugins/smart-tpm-prod/` 下有什么 `skills/`，安装这个 plugin 就能看到什么 skills。`_shared/` 不在任何 plugin 目录内，客户端不认。

要让 3 个变体的用户都能用同一份 skill，必须**物理存在于每个变体目录下**。

### Q：那为啥不用 symlink？

Windows 上 git 处理 symlink 的兼容性不稳：

- 需要管理员权限（开发者模式开了也未必稳）
- 需要 `git config core.symlinks=true`
- 不同 git 版本对 symlink 的 add / checkout 行为不一
- clone 后变成纯文本文件的情况发生过

物理拷贝（`Copy-Item -Recurse`）是最稳的方案，代价是 3 份冗余。本仓库选择"冗余可控 > 工具链兼容性赌"。

### Q：`_shared/skills/about-smart-tpm-mcp/` 这个百科本身要不要复制到 3 变体？

**视需求**。本百科是 `user-invocable: true`，即用户可以 `/skill about-smart-tpm-mcp` 直接调起。

- 如果想让安装 prod 变体的工程师能直接调起百科 → 复制到 `plugins/smart-tpm-prod/skills/about-smart-tpm-mcp/`
- 如果百科只给"看本仓库源码的人"读 → 只放 `_shared/` 也行

当前**建议复制到 3 变体**（沿用其他 9 个 skill 的做法，保持一致性）。

### Q：`marketplace.json` 在根目录的 `.claude-plugin/` 下，`plugin.json` 在每个变体的 `.claude-plugin/` 下，这层 `.claude-plugin` 必须有吗？

是 Claude Code plugin 规范要求的。改名或挪位置会导致客户端识别不出。

---

## 版本

### Q：版本号要不要 semver

形式上是 semver（`major.minor.patch`），但当前节奏偏保守，bug 修和加 skill 都走 patch 升档。

**建议规则**（非强制）：
- 加新 skill / 新 tool 注册 → minor
- 改后端 URL / 兼容性 bug 修 / skill 内容修订 → patch
- 不兼容的契约变化（如 `mcpServers.smart-tpm` key 改名）→ major

### Q：3 个变体的 version 必须严格同步吗

**是**。否则 marketplace 的 `metadata.version` 跟不上、用户 `/plugin update` 时部分变体不更新、CHANGELOG 对应关系断裂。

### Q：marketplace.json 的 metadata.version 跟 plugin.json 的 version 是不是要一致

**是**，当前规则。三者（marketplace.json 的 metadata.version + 3 个 plugin.json 的 version）保持四份同号。

---

## 同步

### Q：改完 `_shared/skills/<x>/` 怎么验证 3 变体都同步了

PowerShell：

```powershell
$src = 'plugins\_shared\skills\<x>'
foreach ($v in 'local','test','prod') {
  $dst = "plugins\smart-tpm-$v\skills\<x>"
  $diff = Compare-Object (Get-ChildItem -Recurse $src) (Get-ChildItem -Recurse $dst) -Property Name,Length
  if ($diff) { Write-Host "❌ $v 不同步"; $diff } else { Write-Host "✅ $v 同步" }
}
```

或更彻底用 `diff -r` (git bash) / `Compare-Object` 跑文件内容。

### Q：能不能写个脚本自动同步

可以。本仓库当前没固化脚本（维护频率低，3 行 Copy-Item 够用），但如果你想加可以放 `scripts/sync-shared.ps1` 之类位置，发版 checklist 里加一条"跑 sync 脚本"。

### Q：误把改动写到了某个变体目录而不是 `_shared/`，怎么救

```powershell
# 反向同步回 _shared
Copy-Item -Recurse -Force plugins\smart-tpm-prod\skills\<x> plugins\_shared\skills\<x>
# 再正向同步到另两个变体
Copy-Item -Recurse -Force plugins\_shared\skills\<x> plugins\smart-tpm-local\skills\<x>
Copy-Item -Recurse -Force plugins\_shared\skills\<x> plugins\smart-tpm-test\skills\<x>
```

下次注意从 `_shared/` 改起。

---

## 兼容

### Q：为啥 `mcpServers.smart-tpm` 既有 `url` 又有 `httpUrl`

Qwen Code CLI 对缺 `httpUrl` 的 server 会**误识别为 SSE transport**，连不上。同时写 `url` + `httpUrl` 且值相同，对 Claude Code / Codex 是冗余（它们只读 `url`），对 Qwen 必要。

详见 [reference/11-ai-integration.md](../reference/11-ai-integration.md)。

### Q：测试新客户端兼容性怎么搞

最小验证 5 步（详见 [reference/11-ai-integration.md](../reference/11-ai-integration.md)）：

1. 客户端能否解析 `plugin.json` 识别 MCP server
2. 能否走完 OAuth 2.1 + PKCE + DCR 流程
3. 调 `tools/list` 能否拿到本插件的全部 tool（约 20 个）
4. 能否成功调一个 read tool
5. 能否加载 skill 文件

跑通 5 步即兼容。新客户端的特殊补丁字段（如 Qwen 的 `httpUrl`）可一并加进 `plugin.json`，与其他客户端不冲突。

### Q：后端新加了 tool，我怎么知道该塞到哪个 skill 的 `allowed-tools`

看 tool 的功能域：

- 数据集相关 → `sample-dataset-extraction` 或新建一个数据集 skill
- 检测流程相关 → 看是查询还是节点能力，分别放 `detect-flow-node-reference` / 新建任务 skill
- 检测记录（生产线实时数据）→ `query-detect-records`
- 测试任务相关 → `create-test-task-from-template` / `reproduce-customer-error`
- 算法链相关 → `locate-test-record-algorithm-chain`

如果新 tool 是全新模块（如 `users_*`），考虑新建一个 skill 而不是塞到现有的。

### Q：怎么知道当前后端实际注册了哪些 tool

调 `tools/list` 是最权威的：

```bash
# 拿 access_token 后
curl -X POST http://192.168.2.175:8080/api/v2/mcp \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

返回的 `result.tools[*].name` 就是真相。

---

## 其他

### Q：仓库要不要加 CI

可以加，但目前没有。能加的检查：

- JSON 格式校验（`marketplace.json` / `plugin.json`）
- 3 变体 version 一致性
- `_shared/skills/<x>/` 与 3 变体目录的 `skills/<x>/` 内容一致性
- README 的 skill 表与实际 `_shared/skills/` 目录一致性

非必要。当前频率下手工 checklist（[scenarios/02](../scenarios/02-maintainer-release-workflow.md) 末尾）也够。

### Q：用户报"装不上"，让我看什么

按这个顺序问用户：

1. 客户端版本（`claude --version` / `codex --version` / `qwen --version`）
2. `claude mcp list` 或等价命令的输出
3. `/plugin list` 的输出
4. 终端有没有 OAuth 跳转的 URL 打印
5. `curl <对应变体的 URL>/.well-known/oauth-authorization-server` 在他机器上的输出

90% 问题在前 3 个就能定位。

# 02 — 维护者发版工作流

> **受众**：本仓库的维护者。日常动作是"加 skill / 改 plugin.json / bump 版本 / push"。
>
> **核心心智**：`_shared/skills/` 是事实源；3 个变体目录是物理拷贝；版本号 3 变体齐步走。

## 仓库布局回顾

详见 [reference/02-architecture.md](../reference/02-architecture.md)。简短复述：

```
plugins/
├── _shared/skills/           ← 改这里（唯一事实源）
├── smart-tpm-local/skills/   ← 物理拷贝
├── smart-tpm-test/skills/    ← 物理拷贝
└── smart-tpm-prod/skills/    ← 物理拷贝
```

每个变体目录还有自己的 `.claude-plugin/plugin.json`（URL 不同，version 同步）。

## 典型场景 A：加一个新 skill

**示例**：新加 skill `query-something-new`。

### Step 1 — 在 `_shared/skills/` 下建目录 + 写 `SKILL.md`

```
plugins/_shared/skills/query-something-new/SKILL.md
```

frontmatter 关键字段：

```yaml
---
name: query-something-new
description: 用户说 X / Y / Z 时触发本 skill。
allowed-tools: tool_a, tool_b, tool_c
---
```

`allowed-tools` 里写的 tool **必须**是 Smart TPM 后端 `tools/list` 已注册的真实 tool 名。写错了 AI 调用时直接失败。

> **核实办法**：本机连任一变体后跑 MCP `tools/list`（或看 Smart TPM 后端的 tool 注册代码），把 frontmatter 里要写的名字逐字对照。改完 frontmatter 后必须 e2e 跑一遍触发该 skill 的对话，确认 AI 能成功调到 tool。

### Step 2 — 物理复制到 3 个变体

PowerShell 一把梭：

```powershell
$src  = 'plugins\_shared\skills\query-something-new'
$dsts = @(
  'plugins\smart-tpm-local\skills\query-something-new',
  'plugins\smart-tpm-test\skills\query-something-new',
  'plugins\smart-tpm-prod\skills\query-something-new'
)
foreach ($d in $dsts) {
  if (Test-Path $d) { Remove-Item -Recurse -Force $d }
  Copy-Item -Recurse $src $d
}
```

> **为什么不用 symlink**：Windows + git + symlink 的兼容性坑太多（详见 [reference/02-architecture.md](../reference/02-architecture.md)）。物理拷贝最稳。

### Step 3 — bump version

3 个变体的 `.claude-plugin/plugin.json` 全部把 `version` 同步 bump（如 `1.0.4 → 1.0.5`）。同时根目录 `.claude-plugin/marketplace.json` 的 `metadata.version` 也跟着 bump。

> **bump 策略**：加 skill / 加 tool 等"新能力"走 minor（`1.0.x → 1.1.0`），bug 修复走 patch（`1.0.4 → 1.0.5`）。当前节奏偏保守，全用 patch 也能接受。

### Step 4 — 更新 README + CHANGELOG

- `README.md` 的 "Skill 一览" 表加新行
- `CHANGELOG.md` 新版本号下记一条 entry

### Step 5 — commit + push

```bash
$ git add plugins/ README.md CHANGELOG.md
$ git commit -m "feat: 加 query-something-new skill (1.0.4 → 1.0.5)"
$ git push
```

用户端下次 `/plugin update` 会拉到新版。

## 典型场景 B：改某个变体的后端 URL

**示例**：测试环境从 121 迁到 122。

只动 `plugins/smart-tpm-test/.claude-plugin/plugin.json` 的 `url` 和 `httpUrl` 两个字段 → bump version（3 变体同步）→ commit + push。

`_shared/skills/` 不动。

## 典型场景 C：改某个 skill 的内容（不改 tool 名）

只动 `_shared/skills/<x>/SKILL.md` → 物理复制到 3 个变体目录（覆盖）→ bump patch version → commit + push。

## 典型场景 D：后端新增 / 改名了一个 tool

1. 找出所有引用该 tool 名的 skill（grep `allowed-tools` 字段）
2. 在 `_shared/skills/` 下改 `allowed-tools` + 可能要改 SKILL.md 正文
3. 物理复制到 3 变体
4. bump version
5. commit + push

## 发版前 checklist

```
[ ] _shared/skills/<改动的 skill>/ 已更新
[ ] 3 个变体目录下的对应 skill 已物理同步（diff -r 验证零差异）
[ ] 3 个变体的 plugin.json version 已 bump 且一致
[ ] marketplace.json 的 metadata.version 已 bump 且与 plugin.json 一致
[ ] README.md 的 Skill 表 / scope 表已更新（如有变化）
[ ] CHANGELOG.md 新版本号下有 entry
[ ] git status 干净（除待提交内容外无杂物）
[ ] 至少在 local 变体上 e2e 跑通新 / 改动的 skill
```

跑一遍 checklist 再 push。

## 不要做

- ❌ 只改 `_shared/`，忘了同步到 3 变体 —— 用户装变体后看不到改动
- ❌ 只改一个变体的 plugin.json version，另两个不动 —— 版本漂移
- ❌ 在 `plugin.json` / `marketplace.json` 里加 `//` 注释 —— Claude Code 严格 JSON 解析，拒绝加载
- ❌ 把不存在的 tool 写进 `allowed-tools` —— 用户调 skill 时 AI 调用失败
- ❌ 用 git symlink 代替物理复制 —— Windows 上不稳

## 验证某个改动是否真生效

最快办法：

```bash
$ cd /tmp/test-install
$ claude
> /plugin marketplace add /path/to/local/SmartTPM_Plugins  # 用本地路径
> /plugin install smart-tpm-local
> /skill
[确认新 skill 在列表里]
> [用自然语言触发新 skill，看 AI 能不能正常调 tool]
```

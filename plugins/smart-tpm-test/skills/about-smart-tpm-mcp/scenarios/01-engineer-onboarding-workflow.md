# 01 — 工程师本地接入工作流

> **受众**：第一次听说本插件、想本地接入并跑通第一个查询的工程师 / 数据 / 算法同事。
>
> **预期耗时**：5-10 分钟（含 OAuth 授权浏览器跳转）。

## 前置检查

| 项 | 怎么自查 |
|---|---|
| 已有 Smart TPM 平台账号 | 能用浏览器登录公司 Smart TPM Web |
| 在公司内网（或 VPN 已连） | `ping 192.168.2.175` 通 |
| 已装 Claude Code（或 Codex / Qwen Code）CLI | 终端跑 `claude --version` 有输出 |
| 知道自己该装哪个变体 | 看 [reference/03-edition-comparison.md](../reference/03-edition-comparison.md) 的决策表，多数日常用 `smart-tpm-prod` |

## Step 1：装插件

```bash
$ claude
> /plugin marketplace add bestfunc/smart_quality_plugins
✓ Marketplace 'smart_quality_plugins' added (3 plugins available)

> /plugin install smart-tpm-prod
✓ Plugin 'smart-tpm-prod' installed
  MCP server 'smart-tpm' needs authorization. Open browser? (Y/n)
```

按 **Y**，下一步走 OAuth。

> **提示**：如果用的是 Codex / Qwen Code，命令略不同，但思路一致 —— 都是 marketplace add + plugin install。

## Step 2：OAuth 授权（浏览器一次同意）

浏览器自动弹出 Smart TPM 登录页 →

```
登录（公司账号）
   ↓
看到 scope 勾选页（具体条目以后端 well-known 为准，下面是典型布局）：
   ☐ mcp:datasets:read         看数据集 / 标注 / 项目标记字典
   ☐ mcp:datasets:write    ⚠️  创建数据集草稿 / 导入样本 / AI 打粗标+细标 (v1.0.10)
   ☐ mcp:detect_flows:read     看检测流程 / 节点 / 执行记录 / 算法流
   ☐ mcp:test_tasks:read       看测试任务 / 测试记录
   ☐ mcp:test_tasks:write      基于模板新建任务草稿
   ☐ mcp:algorithms:read       看算法 / 算法流 / 参数库
   ☐ mcp:detect_records:read   看生产线检测记录(工位分库)

按需勾选 → 点【同意授权】
   ↓
浏览器显示"授权成功，可关闭窗口"
   ↓
终端自动恢复，✓ Authorized.
```

**推荐**：第一次先全勾。后面想撤销随时去 Smart TPM Web → 个人设置 → 已连接的应用。

**不会复制粘贴 token，整个过程零配置。** 原理见 [reference/04-key-concepts.md](../reference/04-key-concepts.md) 的"OAuth 2.1 + PKCE + DCR"段。

## Step 3：验证

```bash
> /mcp
smart-tpm: ✓ connected (61 业务 tool 可用,以 tools/list 为准)

> /skill
quickstart-smart-tpm-mcp            快速上手
smart-tpm-business-concepts         业务概念速查
dataset-fields-reference            数据集字段速查
detect-flow-node-reference          检测流程节点速查
sample-dataset-extraction           抽取数据集样本
locate-test-record-algorithm-chain  定位测试记录算法链路
create-test-task-from-template      基于模板新建测试任务
reproduce-customer-error            复现客户报错
query-detect-records                查检测记录
mark-samples-with-ai                AI 辅助打标 (v1.0.10)
about-smart-tpm-mcp                 本百科
```

两个命令都有期望输出 → 装好了。

## Step 4：跑第一个查询

最容易出效果的，是让 AI 走 quickstart skill。在 Claude Code 里直接说：

```
> 帮我看一下当前可见的数据集前 10 个
```

AI 会自动匹配 `quickstart-smart-tpm-mcp` 或 `sample-dataset-extraction` skill，调 `datasets_list`，几秒内返回结果。

**第一次跑通后的推荐探索路径**：

1. **复盘客户报错**（最有视觉冲击）→ 让 AI 走 `reproduce-customer-error` skill，给它一个产品编号 + 工位 + 时间。v1.0.9 起内部走 `detect_records_trace` 一键拿 3 层链路
2. **算法链路追溯** → `locate-test-record-algorithm-chain`，给它一条 test_record id；现在能看到算法节点级 in/out (`detect_logs_*`)
3. **仿测试任务**（写）→ `create-test-task-from-template`，给它一个模板任务号
4. **AI 辅助打标**（v1.0.10 新加，写）→ `mark-samples-with-ai`，给它一个数据集 + 项目 + 想批量标的 channel 列表；**先在 Web 抽样验证 AI 标的对**，再放开批量

## Step 5：日常使用心智

| 想干啥 | 怎么跟 AI 说 |
|---|---|
| 查数据 | "查一下 X" / "列一下当前 Y" |
| 复盘 | "复现一下客户报的 X 这个错" |
| 仿配置 | "仿 #1024 模板新建一份，X 改成 Y" |
| 批量打标 | "把数据集 X 这 200 个 channel 全标成 OK" / "给这个 channel 加一段 NG 细标 12.5-18.3s" |
| 不知道有啥能查 | "你都能查哪些 Smart TPM 的东西？" |
| 查术语 | `/skill smart-tpm-business-concepts` 直调 |
| 切环境 | 改 `.claude/settings.json` 的 `enabledPlugins` 后缀(`-prod`↔`-test`↔`-local`)→ 重启 |

AI 会**自己挑 skill 和 tool**。你不需要记 tool 名。

## 常见卡点

如果中间任何一步卡住，去 [faq/01-user-questions.md](../faq/01-user-questions.md)，里面分类列了 4 种典型失败模式：装不上 / 授权失败 / tool 不可见 / scope 不够。

## 切换变体（少数场景）

**推荐**：改 `.claude/settings.json` 的 `enabledPlugins` 后缀 → 重启 Claude Code → 重新走 OAuth。

```json
// .claude/settings.json
"enabledPlugins": {
  "smart-tpm-prod@smart_quality_plugins": true   // 改成 -test / -local
}
```

不用 uninstall + install 一遍 plugin 文件,只是切启用状态。

**不要同时启用两个变体**：三变体的 MCP server key 都是 `smart-tpm`，会互相覆盖（见 [reference/03-edition-comparison.md](../reference/03-edition-comparison.md)）。

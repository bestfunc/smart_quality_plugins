# 03 — 销售 / CS 演示工作流

> **受众**：要给客户 / 内部高层 / 行业活动现场演示"我们的平台支持 AI 直接驱动"的人。
>
> **目标**：在 5-10 分钟内让观众"哇"一声，理解三件事：
> 1. AI 不是聊天机器人，是真的能调我们后端
> 2. 跨页面跨模块的复杂操作被压成一句话
> 3. 权限可控、安全可审计，不是黑箱

## 演示前准备（必做）

| 项 | 说明 |
|---|---|
| **装 `smart-tpm-test` 变体** | **不要用 prod** —— 现场误操作影响真实运营 |
| 提前在 test 环境 seed 几条"客户报错"样本 | 演示"复现客户报错"时有数据可演 |
| 提前打开浏览器登好 Smart TPM Web | 演示完 AI 操作后可切到 Web 印证"AI 干的事真的发生了" |
| 终端字体调大（至少 18pt） | 投屏可读 |
| 网络稳定 | OAuth 跳转 + MCP 调用都吃网络 |
| 演示账号已授权全部 scope | 避免现场卡在 scope 不足 |

## 推荐演示顺序（10 分钟版）

### 开场（1 分钟）

```
"我们的 Smart TPM 平台有完整的 Web 操作界面，今天给你看的是另一种方式 —
工程师在终端用自然语言跟 AI 说话，AI 直接调我们后端干活。"
```

打开终端，让观众看到 Claude Code 已就绪。

### Demo 1：开口即查（2 分钟）

```
> 帮我列下当前组织下的数据集，按修改时间倒序，前 5 个
[AI 走 datasets_list → 几秒内返回表格]
```

**话术钩子**：
> "注意这里没有任何 SQL、没有任何配置文件、没有任何 API 文档查阅 —— 工程师就一句话，AI 自己挑工具、填参数、解析结果。"

### Demo 2：跨模块编排（3 分钟）

最有冲击力的一个 ——

> ⚠️ **演示前必做**：下方 prompt 里的 `<产品编号>` / `<工位>` / 日期是占位符。**直接照抄会拿到空结果**。开场前 5 分钟用 Web 翻到对应记录确认数据存在,再把占位符替换成你在 test 环境 seed 好的真实样本值。

```
> 给我复现一下：产品编号 <你 seed 的真实编号> 在 <工位> 2026-05-20 报的 NG，
> 串出它走过的检测流程节点 + 每个节点用的算法版本

[AI 自动走 reproduce-customer-error skill，串 4 个 tool：
  test_records_get → detect_flows_get → algorithm_flows_get → algorithms_get]
```

**话术钩子**：
> "这一句话在 Web 界面上要工程师跳 4 个页面、人工抄数据。AI 这里是**自己编排**的 —— 它知道哪个 tool 拿什么、需要什么顺序、要把什么 ID 串起来。"

切回浏览器 Smart TPM Web，打开对应 test_record 印证 AI 给的数据准确。

### Demo 3：可控的写操作（2 分钟）

```
> 仿造测试任务 #1024 的模板新建一份，sampleRatio 改成 0.3
[AI 走 create-test-task-from-template skill →
  test_tasks_get(#1024) → test_tasks_create_from_template → 返回新任务草稿 ID]
```

**话术钩子**：
> "整个约 20 个 tool 里只有这 1 个写操作。其他全是只读。即便是这个写，也只是创建草稿 —— 真正生效还要人在 Web 里点确认。AI **不能**改你们的生产配置、不能动检测流程、不能改算法参数。"

### Demo 4：权限与审计（2 分钟）

打开 Smart TPM Web → 个人设置 → 已连接的应用：

```
[Web 页面显示]
应用：Claude Code (smart-tpm-test)
已授予 scope：
  ✓ mcp:datasets:read
  ✓ mcp:detect_flows:read
  ✓ mcp:test_tasks:read
  ✓ mcp:test_tasks:write
  ✓ mcp:algorithms:read
  ✓ mcp:detect_records:*
最近调用：12 分钟前
[撤销授权] 按钮
```

**话术钩子**：
> "客户每次授权时**自己勾选**给 AI 哪些权限。所有调用都落审计日志，超管在系统设置里能看到完整 trace。AI 不是黑箱 —— 它的每一步都跟人在 Web 上点的按钮一样可追溯。"

## 客户最常问的 3 个问题

### Q1：跟你们 Web 界面比，强在哪？

| 维度 | Web | AI + MCP |
|---|---|---|
| 单一查询 | ⭐⭐⭐ 鼠标点几下 | ⭐⭐ 同等量级 |
| 跨页面编排 | ⭐ 多个 tab 来回切 | ⭐⭐⭐⭐⭐ 一句话搞定 |
| 批量复盘 | ⭐ 一条一条点 | ⭐⭐⭐⭐⭐ "复现这 20 条客户报错" |
| 不熟悉系统的人 | ⭐⭐ 学习曲线 | ⭐⭐⭐⭐ AI 知道哪个能力在哪 |
| 写操作 | ⭐⭐⭐⭐⭐ 完整 | ⭐⭐ 只 1 个入口（故意） |

**核心论点**：Web 是给"知道自己要点哪"的人；AI 是给"知道自己要什么结果，但不想记界面在哪"的人。**两者互补**。

### Q2：我们已经在用 Qwen Code（或别的 AI CLI），能接吗？

**能**。本插件用 MCP 标准协议 + HTTP transport，任何符合 MCP 规范的客户端都能接。Qwen Code 已验证可用（1.0.2 起兼容）。Codex 也兼容。详见 [reference/11-ai-integration.md](../reference/11-ai-integration.md)。

### Q3：AI 会不会改坏我们的数据？

**当前设计上不会**：

- 约 20 个 tool 里只有 **1 个写操作**（基于模板新建测试任务草稿），且**生成的是草稿**，要在 Web 里人工确认才生效
- `detect_flows` 模块**完全没有写权限**（故意的，防 AI 改坏生产流程）
- 每次操作都需要对应 OAuth scope，**用户授权时可只勾读权限**
- 所有调用都落审计日志（`mcp_audit_log` 表），超管可在 Web → 系统设置 → MCP 审计 看到完整 trace

如果客户对此还有顾虑，引导到 [reference/04-key-concepts.md](../reference/04-key-concepts.md) 的 Scope 段做详细解释。

## 不要在客户面前做的事

- ❌ 用 `smart-tpm-prod` 演示 —— 真实数据 + 真实写权限，现场误操作影响运营
- ❌ 演示需要复杂中间数据的场景而没提前 seed —— 现场 0 结果尴尬
- ❌ 临场猜测 tool 名称 —— AI 自己挑，不要替它解释你不确定的 tool
- ❌ 承诺未来功能 —— roadmap 用弱措辞（详见 [reference/08-roadmap.md](../reference/08-roadmap.md)）
- ❌ 演示出错时尝试"现场调试" —— 跳过去演别的 demo，事后再排查

## 演示完的引导话术

```
"想自己试一下的同学，3 行命令装好：
  /plugin marketplace add bestfunc/smart_quality_plugins
  /plugin install smart-tpm-test     # 测试环境
  [浏览器授权]
完事。装完默认就有 9 个 skill，问 AI '你都能查啥' 就行。"
```

进一步的内部话术（**不**对外发）见 [faq/03-sales-talking-points.md](../faq/03-sales-talking-points.md)。

# 03 · 核心概念（DAG / 节点 / 三级参数 / mode / 热加载）

> 事实来源：`v2/dag/`、`docs/AI_Dag_Load.md`、`README.md`。改算法/加节点必读。

## DAG 流程引擎

前端流程编辑器导出的不是 BPMN，而是顶层 `{nodes, edges}` JSON：

- **node** 从 `data` 下取 `classId` / `params` / `mode` / `enabled`
- **edge** 从 `source` / `target` 取值，条件从 `data.condition` 取（空字符串转 `None`）

引擎核心三件套：

| 模块 | 职责 |
|---|---|
| `DAGEngine` | 解析 JSON、构建拓扑、检测循环依赖、校验合法性，`step_in`/`step_out` 由拓扑自动推断 |
| `DAGExecutor` | 按拓扑驱动执行：顺序链 / Fork 并发 / Join 合并 / 条件路由（`_condition`） |
| `DAGFlowAdapter` | 对外适配器，提供与原 Flow 引擎一致的 `start()` / `result` 接口 |
| `NodeRegistry` | 节点注册表，`class_id` → 纯函数映射，支持热插拔；执行器优先用 `class_id` 查找，`enabled=False` 跳过 |

## 节点契约（9 类）

每个节点遵守 `step_in → step_out` 契约（输入分支数 → 输出分支数）：

| class_id | 节点 | step_in → step_out | 说明 |
|---|---|---|---|
| `start` | 启动 | zero → one | 读取音频文件，初始化上下文 |
| `data_check` | 数据校验 | one → n | 校验音频时长/分贝，按后继数量复制输出 |
| `data_split` | 音频切分 | one → n | 6 种切分策略（自动 / 按时间 / 按曲线 / 按分类） |
| `ai_test` | AI 推理 | one → n | `mode` 动态分发到对应算法模型 |
| `join_step` | 结果合并 | n → one | 合并多个分支的推理结果 |
| `on_off_step` | 结果过滤 | one → one | 按曲线数据过滤无效时间区间 |
| `classify` | 条件分类 | one → one | 阈值判定 `ok`/`ng`，驱动条件边路由 |
| `sub_step` | 子流程 | one → one | 嵌套执行独立的 DAG 子流程 |
| `end` | 结束 | one → one | 提取 `ai_result` 作为最终输出 |

> **契约红线**（来自 git log / CLAUDE.md）：`split` / `data_check` 单后继时返回单对象（dict），不要包成 list，否则 `_validate_output` / `step_out` 校验会误报。详见 [faq/01-developer-questions.md](../faq/01-developer-questions.md)。

## 三级参数解析（NodeParamResolver）

```
优先级：前端 > 数据库 > 本地 JSON

L1 前端参数 (data.params)              ← 用户在编辑器配置
   ↑ 覆盖
L2 数据库参数 (flow_node_param_base)   ← 管理员维护
   ↑ 覆盖
L3 本地 JSON (models/{class_name}/*.json) ← 出厂默认值
```

- **同名 key**：高优先级覆盖低优先级
- **不同 key**：各层独有参数全部保留（合并而非替换）
- `DAGEngine` 支持 `param_resolver` 注入；无 resolver 时向后兼容（只用前端参数）

## mode 动态分发

`ai_test` 节点的 `mode` 参数（拼成 `model_type`）决定本次推理调用哪个算法模型：`mode` → `model_info`（flow_engine 按 mode/version 从 DB 注册的制品条目）→ `class_configs`（class_name → class_manager 模块）→ 动态导入并 `getattr(module, "Manager")` 实例化。这让同一套 DAG 结构能驱动不同算法，无需为每个算法写独立节点。（注：`setting.yaml` 里的 `model_configs` 是另一回事，仅供三级参数解析查本地 JSON 默认值用，不在 Manager 分发链上；`ModelAdmin` 上并无 `model_configs` 属性。）

> 注意区分两个 `mode`：
> - **节点级 `mode`**（DAG 内）：选算法模型
> - **请求级 `mode`**（`/detect/execute` 入参）：`"0"`=实际结果 / `"1"`=全 OK / `"2"`=全 NG（调试用）

## 热加载 / 延迟加载

| 机制 | 说明 |
|---|---|
| **热加载** | `POST /api/v2/detect/reloadOne`（或 `flow/reloadOne`）不停机加载新流程/模型；集群下 master 接收后广播到所有 worker |
| **延迟加载** | 模型支持首次调用时才加载，减少启动时间；子流程懒加载——状态为 draft 也会自动尝试加载 |
| **冷下载耗时** | 首次 reload 需从 MinIO 冷下载模型，30s~3min，客户端 timeout 要 ≥ 120s |

> 示例：管理员在后台改了某算法节点的阈值 → 发布流程 → 调 `reloadOne` → 集群所有 worker 级联加载新版本，期间服务不中断。

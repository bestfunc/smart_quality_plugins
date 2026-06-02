# 场景 01 · 开发新人上手

> 受众：刚接手代码的开发。目标：本地跑起来 → 跑测试 → 知道怎么加节点 / 加算法。

## 工作流：从 0 到能改代码

### 1. 起本地环境

```bash
conda create -n venv python==3.10.12 && conda activate venv
pip install torch --index-url https://download.pytorch.org/whl/cu121
pip install -r v2/requirements.txt
mysql -u root -p < docs/sql/dag_branch_0.1.sql      # 初始化 DB
python v2/server_start.py                       # 启动
```

> 本机坑（CLAUDE.md 全局）：命令名 `python`（非 `python3`）；含中文脚本写 `.py` 文件别用 heredoc。

### 2. 跑测试（改 DAG/参数必跑；用例总数随测试增长，以 `pytest tests/ --collect-only -q` 实际收集数为准）

```bash
python -m pytest tests/ -v                              # 全部
python -m pytest tests/test_dag_engine.py -v            # DAG 引擎（36）
python -m pytest tests/test_dag_param_resolver_complex.py -v -s   # 参数解析（含报告）
```

| 测试文件 | 覆盖 |
|---|---|
| `test_dag_engine.py` | 解析/拓扑/条件路由/循环检测/校验 |
| `test_dag_complex_model.py` | 多模型动态调用、mode 透传、enabled 跳过 |
| `test_dag_param_resolver*.py` | 三级参数回退、8 种组合穷举 |

### 3. 理解改动落点

| 想改什么 | 去哪 | 先读 |
|---|---|---|
| 加 DAG 节点类型 | `v2/dag/nodes/node_*.py` + 在 `dag_node_registry.py` 注册 | [reference/03-key-concepts.md](../reference/03-key-concepts.md) |
| 加算法模型 | `v2/models/BF_*/` + `model_admin.py` 注册 | [scenarios/03-add-algorithm-flow.md](./03-add-algorithm-flow.md) |
| 改检测引擎/聚合 | `v2/detect/` | [reference/04-detect-engine.md](../reference/04-detect-engine.md) |
| 加 API 端点 | `v2/api/*_api.py` | 用 `add-endpoint` skill |

## 示例：加一个 DAG 节点的最小路径

1. 在 `v2/dag/nodes/` 写节点纯函数，遵守 `step_in → step_out` 契约（单后继返回 dict 不要包 list）
2. 在 `dag_node_registry.py` 用 `class_id` 注册
3. 写本地 JSON 默认参数 `models/{class_name}/*.json`（三级参数 L3）
4. 加测试到 `tests/test_dag_*.py`，`pytest` 通过
5. 提交走 `dev` 分支（分支约定见 `CLAUDE.md`）

> 红线提醒：节点新增异常路径要先写 `ctx.node_logs` 的 fail 条目再 raise（保证可追溯），详见 [faq/01-developer-questions.md](../faq/01-developer-questions.md)。

# 02 · 整体架构

> 事实来源：`README.md`、`docs/product_architecture_doc.md`、`v2/api/`、`v2/dag/`。

## 定义：分层架构

系统从上到下分为五层，请求自唯一对外入口 nginx 进入，最终落到 PyTorch 算法模型层：

```
外部业务系统（ERP / MES / MASS）
        │ HTTP POST (JSON)，X-API-Key 鉴权
        ▼
nginx 反代（唯一对外入口，端口见集群配置）
        │ 按路径分流
        ▼
FastAPI 服务层（v2/api/）
   server.py · detect_api.py(v2/detect/) · flow_api.py · dashboard_api.py · cluster_api.py
        │
   ┌────┴───────────────┬──────────────────┐
   ▼                    ▼                  ▼
检测引擎(DetectEngine)  DAG 引擎/执行器     参数解析器
解析通道+并行调度       拓扑排序/分支合并    三级回退
        │
        ▼
节点注册表(NodeRegistry)：class_id → 节点函数
   start · data_check · data_split · ai_test · join_step
   · on_off_step · classify · sub_step · end
        │
        ▼
模型管理层(ModelAdmin)：mode → model_info → class_configs → 动态加载 Manager
        │
        ▼
算法模型层（21+ BF_* 算法，PyTorch / NumPy / SciPy）
        │
   ┌────┴────┬──────────┐
   ▼         ▼          ▼
 MySQL     MinIO      Qdrant
流程/日志   模型/音频   特征向量
```

> 注：上图服务层列的是真正注册到 FastAPI 的路由文件——`dashboard_api` / `cluster_api` / `flow_api` 在 `v2/api/`，`detect_api`（含 `/api/v2/detect/execute`）在 `v2/detect/`。`v2/api/model_api.py` **不暴露 HTTP 路由**，它是模型扫描 / 版本注册与 DB 同步模块，被 `server.py` 以 `import v2.api.model_api as model_manager` 方式导入。

## 细节：各模块职责

| 模块 | 职责 | 技术栈 |
|---|---|---|
| **API 网关层** | 接收外部 REST 请求、鉴权、路由分发、负载均衡转发 | FastAPI + Uvicorn |
| **Web 管理端** | 流程编辑、模型管理、集群监控、日志查看、用户管理 | Python Dash |
| **检测引擎** | 解析检测请求、匹配流程配置、并行调度算法、聚合结果、输出判定 | Python 多线程 |
| **流程引擎(FlowEngine)** | 算法子流程的加载、缓存、版本切换、热更新 | 单例 + DB 驱动 |
| **DAG 执行器(DAGExecutor)** | 将 `{nodes, edges}` 解析为 DAG，按拓扑执行（顺序/Fork/Join/条件路由） | 拓扑排序 + 线程池 |
| **模型管理(ModelAdmin)** | PyTorch 加载 / GPU 推理 / 从 MinIO 下载制品 / 版本注册 | PyTorch + CUDA |
| **集群管理** | Master-Worker，心跳检测，文件同步，负载均衡 | HTTP + 自定义协议 |

## 示例：一次请求的完整链路

```
客户端 POST /api/v2/detect/execute
  → nginx 按路径分流到 ai_workers（w1/w2/w3，round-robin）
  → FastAPI 路由 + X-API-Key 鉴权（detect_engine:execute）
  → DetectEngine 解析 dataContent 五张表 → 匹配 stageAssignments
  → 从 MinIO 下载音频文件
  → 按 stage 并行（线程上限 DETECT_MAX_WORKERS：生产 8、代码默认 16）调度算法子流程：
       NodeParamResolver（三级参数合并）
       → DAGEngine（解析 JSON、拓扑排序、循环检测、校验）
       → DAGExecutor（顺序 / Fork并发 / 条件路由 / Join合并）
       → 各节点函数（NodeRegistry 查 class_id，ai_test 按 mode 分发模型）
  → 聚合检测结果（point_first 按点位 + stage_first 按工位）
  → JSON 响应同步返回；执行日志异步写入 MySQL 三表
```

> **关键设计点**：检测引擎（DetectEngine）是"业务编排层"，处理多点位/多工位/多阶段；DAG 引擎是"算法编排层"，处理单条算法子流程内部的节点执行。两者是上下游关系——检测引擎为每个 stage 调用一次 DAG 子流程。

数据流、聚合视图详见 [04-detect-engine.md](./04-detect-engine.md)；节点机制详见 [03-key-concepts.md](./03-key-concepts.md)；集群分流详见 [06-deployment-cluster.md](./06-deployment-cluster.md)。

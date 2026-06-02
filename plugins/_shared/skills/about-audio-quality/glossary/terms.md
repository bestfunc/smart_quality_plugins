# 术语表

> AI 音频质检系统专有名词与缩写。按主题分组，首次在其他文件出现的术语都应收录于此。

## 流程引擎

| 术语 | 含义 |
|---|---|
| **DAG** | 有向无环图（Directed Acyclic Graph），前端流程编辑器导出的 `{nodes, edges}` 结构，引擎拓扑排序后执行 |
| **DAGEngine** | DAG 解析器：构建拓扑、检测循环、校验、自动推断 `step_in`/`step_out` |
| **DAGExecutor** | DAG 执行器：顺序 / Fork 并发 / Join 合并 / 条件路由 |
| **DAGFlowAdapter** | 对外适配器，提供与原 Flow 引擎一致的 `start()`/`result` 接口 |
| **NodeRegistry** | 节点注册表，`class_id` → 节点纯函数映射 |
| **FlowEngine** | 流程引擎，管理算法子流程的加载/缓存/版本/热更新 |
| **step_in / step_out** | 节点输入/输出分支数契约（zero/one/n） |
| **`_condition`** | 条件边字段，驱动 classify 后的 ok/ng 路由；空字符串视为 None |

## 节点类型（class_id）

| 术语 | 含义 |
|---|---|
| **start** | 启动节点，读音频、初始化上下文 |
| **data_check** | 数据校验（时长/分贝） |
| **data_split** | 音频切分（6 种策略） |
| **ai_test** | AI 推理节点，按 `mode` 分发算法模型 |
| **join_step** | 多分支结果合并 |
| **on_off_step** | 按曲线区间过滤无效时间段 |
| **classify** | 阈值判定 ok/ng，驱动条件路由 |
| **sub_step** | 子流程，嵌套执行独立 DAG |
| **end** | 结束节点，提取 `ai_result` |

## 参数与调度

| 术语 | 含义 |
|---|---|
| **三级参数解析 / NodeParamResolver** | 前端(data.params) > 数据库(flow_node_param_base) > 本地 JSON，逐级回退 |
| **mode（节点级）** | `ai_test` 选择算法模型的参数 |
| **mode（请求级）** | `/detect/execute` 入参：`0`实际 / `1`全OK / `2`全NG（调试） |
| **热加载 / reloadOne** | 不停机加载新流程/模型，集群下 master 广播到 worker |
| **延迟加载 / 懒加载** | 模型首次调用时加载；draft 子流程自动尝试加载 |

## 检测引擎

| 术语 | 含义 |
|---|---|
| **DetectEngine** | 检测引擎，业务编排层，解析通道+并行调度算法+聚合判定 |
| **dataContent** | 请求体中的五张业务表数据（detect / detect_point / detect_channel / file / detect_station_stage） |
| **flowNumber / versionCode** | 检测流程编号 / 版本号 |
| **stageAssignments** | 流程配置中点位与工位阶段的分配关系 |
| **disabledRules** | 测试任务下发的禁用规则，覆盖 flowData 的 disabled（v0.13.7+） |
| **point_first** | 按检测点位优先聚合的判定视图 |
| **stage_first** | 按工位阶段优先聚合的判定视图 |
| **productResult** | 产品级总判定：positive(合格) / negative(不合格) |
| **stageKey** | stage 唯一标识 `pointID__stageN` |
| **status** | stage 执行状态：success / fail / algorithm_error / cached / skipped |
| **warnings** | 响应中的诊断数组，正常流程不出现 |
| **软失败 / _detect_soft_failure** | 检测算法异常被吞为 success 的防护，须查嵌套 `ai_result.code` |
| **detect_log 三表** | 只写不改不删的执行审计表 |

## 算法模型（BF_*）

| 术语 | 含义 |
|---|---|
| **BF_*** | 算法编号前缀（BingFeng/宝适音频算法系列） |
| **BF_V14A/V15A/V16A** | ResNet+ViT 音频分类网络 |
| **BF_SC_V1~V6** | 多维评分（dB/RMS/FFT） |
| **BF_DB_V1** | 分贝测量 |
| **BF_PS_V1** | 线轮音/脉冲检测（FFT 峰值 + 频谱增强） |
| **BF_FE_V1/V2 / BF_FE_AD_V2** | 特征提取 / 特征异常检测（用 Qdrant） |
| **BF_CF_V1** | 感知评估 |
| **BF_JUDGE_V1** | 多算法仲裁判定 |
| **ModelAdmin** | 模型管理器，动态/延迟加载、线程安全 |
| **模型制品** | MinIO 中的 `MOD-xxxx/Vx.x/`，含 model.pt + config.yaml（必须配套） |
| **preprocess_config** | 预处理参数（n_fft/hop_length/num_mel_bins/target_length），不可业务改 |

## 基础设施

| 术语 | 含义 |
|---|---|
| **MinIO** | S3 兼容对象存储，存模型制品 + 音频 |
| **Qdrant** | 向量数据库，特征相似度检索（仅特征类算法） |
| **ClearML** | MLOps 平台，模型训练追踪与制品管理 |
| **master / worker** | 集群角色；1 master + 4 worker（w0~w3 各占 1 GPU） |
| **ai_v2_lb** | nginx 反代容器，唯一对外入口 |
| **GPU 推理闸门 / GPU_INFER_MAX_CONCURRENT** | GPU 全局并发限制，worker 设 8（禁 0），master 设 0 |
| **ConcurrencyMonitor** | 并发打点监控，日志看 `now=/peak=` 判断是否 hang |
| **round-robin / least_conn** | LB 策略；本系统用 round-robin（禁 least_conn） |
| **DETECT_MAX_WORKERS** | 单请求内 stage 并行线程上限（生产 8） |

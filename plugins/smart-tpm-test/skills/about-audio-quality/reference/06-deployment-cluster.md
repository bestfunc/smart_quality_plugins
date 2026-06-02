# 06 · 部署形态 · 4 GPU 集群 · 发布运维

> 事实来源：`docs/prod_deploy_guide.md`、`docs/multi_gpu_cluster_plan.md`、`CLAUDE.md`、`config_cluster/docker-compose.cluster.yaml`（生产部署后位于 `/home/bestfunc/ai_api_server_v2/docker-compose.cluster.yaml` 根目录）。
> **权威配置以 `docs/prod_deploy_guide.md` 与 `CLAUDE.md` 红线表为准，本文与之冲突时以它们为准。**

## 定义：三种部署形态

| 形态 | 说明 | 镜像 |
|---|---|---|
| 单机 standalone | `AI_CLUSTER_ROLE=standalone`，单容器跑全部 | `Dockerfile_v2.cuda` |
| 多 GPU 集群 | 1 master + N worker + nginx 反代，自动负载均衡 | 同上，role 区分 |
| 边缘 Jetson | Jetson 专用镜像 | `Dockerfile.jetson` |

均基于 Docker；业务代码与模型通过卷挂载注入（不重建镜像即可更新）。

## 细节：生产 4 GPU 集群结构

| 组件 | 容器名 | 角色 |
|---|---|---|
| Master | `ai_v2_master` | 接收 reload/cluster 请求，广播到 worker，不跑算法 |
| Worker × 4 | `ai_v2_w0` ~ `ai_v2_w3` | 各独占 GPU 0~3，承接 detect 推理 |
| 负载均衡 | `ai_v2_lb` | nginx 反代，唯一对外入口 |
| Web 前端 | `ai_v2_prod_web` | Dash 管理后台 |

> ⚠️ **红线（CLAUDE.md）**：
> - **GPU 0 被宿主机其他容器占用**，`ai_v2_w0` 已在 nginx 摘除（`down`），**禁止改回 active**
> - **禁止触碰**老容器 `ai_v2` / `ai_v2_dev` / `ai_v2_dev1` 等
> - `AI_SERVER_WORKERS` **必须保持 `"1"`**（多 worker 触发 `model.pt` rename race）
> - LB 策略用 **round-robin**，禁用 `least_conn`（串行请求下退化为永远选第一个 upstream）

### nginx 路由分流

| 路径 | 上游 | 说明 |
|---|---|---|
| `/api/v2/detect/execute` | `ai_workers`（w1/w2/w3 round-robin） | detect 推理 |
| `/api/v2/detect/reloadOne` `reloadAll` | `ai_master` | master 接收后广播 4 worker |
| `/api/v2/flow/reload*` | `ai_master` | 同上 |
| `/api/v2/cluster/*` | `ai_master` | 集群管理 |
| `/api/v2/*/_internal/*` | **403 拒绝** | 仅集群内部互调 |
| 其他默认 | `ai_master` | 流程查询、配置 |

### 关键 env（CLAUDE.md 速查）

```yaml
AI_SERVER_WORKERS: "1"            # ❗必须 1
DETECT_MAX_WORKERS: "8"
GPU_INFER_MAX_CONCURRENT: "8"     # ❗worker 禁止设 0；master 设 0（不跑算法）
CONCURRENCY_MONITOR_ENABLED: "1"
WEB_CONFIG_FILE: /ai/config/web_config.worker{N}.yaml   # 各 worker 不同
```

## 示例：发布运维三件套

1. **scp 代码**（仅 `v2/` 下文件）→ 4 worker + master 共享挂载，scp 一次即可
2. **md5 校验** 远端 vs 本地一致
3. **滚动重启**：`docker restart ai_v2_master ai_v2_w0 ai_v2_w1 ai_v2_w2 ai_v2_w3`（不动 lb / prod_web）
4. **等 4/4 healthy**：`curl <入口>/api/v2/cluster/workers` 看 `count.healthy == 4`

| 改动类型 | 生效方式 |
|---|---|
| 改 `v2/` 代码 | scp + 滚动重启 |
| 改 `nginx.conf` | scp + `docker exec ai_v2_lb nginx -t && nginx -s reload`（秒级，不重启容器） |
| 改 compose / env | **必须** `docker compose -p ai_v2_cluster -f docker-compose.cluster.yaml up -d`（`restart` 不重读 environment） |

> 生产服务器地址、SSH 方式、DB/MinIO 连接见 `CLAUDE.md` 环境速查表与 `docs/prod_deploy_guide.md`（含真实内网 IP）。本百科不复制真实凭据。完整排障见 [scenarios/02-ops-deploy-troubleshoot.md](../scenarios/02-ops-deploy-troubleshoot.md)。

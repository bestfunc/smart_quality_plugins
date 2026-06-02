# 场景 02 · 运维发布与排障

> 受众：负责发布和线上排障的运维。目标：标准发布 → 四类故障的排障第一步。
> **权威以 `docs/prod_deploy_guide.md` + `CLAUDE.md` 红线表为准。**

## 工作流：标准发布（dev → master → 生产）

1. dev 自测稳定 → `git checkout master && git merge dev` → `git tag vX.X.Y-slug` → `git push --tags`
2. **scp 代码**（仅 `v2/` 下文件）到生产 `/home/bestfunc/ai_api_server_v2/v2/`（4 worker + master 共享挂载，一次即可）
3. **md5 校验** 远端 vs 本地
4. **滚动重启**：`docker restart ai_v2_master ai_v2_w0 ai_v2_w1 ai_v2_w2 ai_v2_w3`（不动 lb / prod_web）
5. **等 4/4 healthy**：`curl <入口>/api/v2/cluster/workers` → `count.healthy == 4`

| 改动 | 生效方式 |
|---|---|
| `v2/` 代码 | scp + 滚动重启 |
| `nginx.conf` | scp + `docker exec ai_v2_lb nginx -t && nginx -s reload`（秒级） |
| compose / env | `docker compose -p ai_v2_cluster -f docker-compose.cluster.yaml up -d`（`restart` 不重读 env） |

> 红线：禁碰老容器（`ai_v2`/`ai_v2_dev*`）；只 scp 到 `…/ai_api_server_v2/v2/`；`AI_SERVER_WORKERS` 必须 1；`ai_v2_w0` 保持 down；LB 用 round-robin。算法 AI 改 `v2/models/**`、`v2/dag/**` 可直接在 master 迭代（见 `docs/algo_ai_deploy_guide.md`）。

## 示例：排障第一步（CLAUDE.md）

| 现象 | 排查顺序 |
|---|---|
| **请求超时（499/5xx）** | ① `curl .../cluster/workers` 看 4/4 → ② `nvidia-smi` 看 4 卡 → ③ `docker logs --since 5m ai_v2_w{0,1,2,3} \| grep ConcurrencyMonitor \| tail -5` 看 `now=N peak=N` 数字是否在变（不变=hang） |
| **节点 hang 死** | `py-spy dump --pid <host-pid>`（宿主跑），找 `_branch_with_track` 在 `acquire` 排队的线程数 |
| **算法 ERROR 但 status=success** | 检查 `_detect_soft_failure` 是否在查嵌套 `ai_result.code`（不查会把异常吞为 success） |
| **reloadOne 499** | 客户端超时太短（默认 5s 不够），冷下载 30s~3min，客户端 timeout 调到 ≥ 120s |

## 其它常见报错（product_architecture_doc §7.3）

| 报错 | 规避 |
|---|---|
| `检测流程配置未找到` | 后台发布流程后调 reloadOne |
| `检测结果全为空（matchCount=0）` | 看响应 `warnings`，多半是 stageAssignments 的 pointID 与请求数据不匹配 |
| 多 worker 数据不一致 | reloadOne 只更新了一个 → 系统有自动同步，仍不一致则滚动重启 |
| `dataContent JSON 解析失败` | 用 JSON body（无大小限制），别用超大 form-data |

> 运维高频问答见 [faq/02-ops-questions.md](../faq/02-ops-questions.md)。

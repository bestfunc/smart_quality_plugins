# FAQ 02 · 运维高频问题

> 受众：运维 / 实施。事实来源：`CLAUDE.md`、`docs/prod_deploy_guide.md`、product_architecture_doc §7。

## 发布与生效

**Q：改了 env / compose，`docker restart` 为什么没生效？**
A：`restart` **不重读 environment**。改 env 必须 `docker compose -p ai_v2_cluster -f docker-compose.cluster.yaml up -d`（必要时 `--force-recreate`）。

**Q：改 nginx 要重启容器吗？**
A：不用。scp 后 `docker exec ai_v2_lb nginx -t && nginx -s reload`，秒级生效。

**Q：scp 代码要传几次？**
A：一次。4 worker + master 共享挂载 `…/ai_api_server_v2/v2/`。传完滚动重启 5 个容器（不动 lb / prod_web），等 `cluster/workers` 的 `count.healthy == 4`。

## reload 与一致性

**Q：reloadOne 返回 499？**
A：客户端超时太短（默认 5s）。服务端冷下载模型 30s~3min，客户端 timeout 调到 ≥ 120s。

**Q：多 worker 数据不一致？**
A：master 接 reload 后会广播到所有 worker。若仍不一致，滚动重启 5 个容器。

## GPU 与并发

**Q：为什么 `ai_v2_w0` 是 down？能改回来吗？**
A：**不能**。GPU 0 被宿主机其他容器占用 ~15GB，w0 已在 nginx 摘除，detect 流量走 w1/w2/w3（GPU 1/2/3）。这是红线。

**Q：能把 `AI_SERVER_WORKERS` 调大提并发吗？**
A：不能，**必须保持 `"1"`**。多 worker 触发 `model.pt` rename race。提并发靠集群横向扩展 + `GPU_INFER_MAX_CONCURRENT`（worker 设 8，禁止 0）。

**Q：LB 为什么不能用 least_conn？**
A：串行请求下 `least_conn` 退化为永远选第一个 upstream，必须 round-robin。

## 排障与日志

**Q：请求大面积超时先查什么？**
A：① `curl .../cluster/workers` 看 4/4 → ② `nvidia-smi` 看 4 卡 → ③ grep `ConcurrencyMonitor` 看 `now=/peak=` 数字是否在变（不变=hang）。详见 [scenarios/02-ops-deploy-troubleshoot.md](../scenarios/02-ops-deploy-troubleshoot.md)。

**Q：detect_log 三表能改吗？**
A：**只写不改不删**，是审计表，禁止 UPDATE / DELETE 历史记录。改 schema 先在测试库验证再上生产，DDL 记入记忆 `sql_migrations`。

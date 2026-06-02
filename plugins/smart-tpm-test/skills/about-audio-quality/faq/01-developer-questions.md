# FAQ 01 · 开发高频问题

> 受众：开发。事实来源：`v2/dag/`、git log、`CLAUDE.md`。

## 节点契约

**Q：`split` / `data_check` 节点单后继时返回什么？**
A：返回单对象（dict），**不要**包成 list。单后继 unwrap 返回 dict，否则 `_validate_output` / `step_out` 校验会误报（git log `0788283` / `66897d8`）。多后继才按后继数量复制输出。

**Q：节点加了异常分支怎么保证可追溯？**
A：在 `_log_node_execution` 写一条 fail 的 `ctx.node_logs` 条目，**再** raise。否则 detect_log_node 表没有 partial 记录，排障看不到中间态（CLAUDE.md 红线）。

**Q：`enabled=False` 的节点会执行吗？**
A：不会。执行器执行节点前先检查 `node_def.enabled`，禁用节点直接把上游输入透传给下游、并写一条 `status="skipped"` 的 `node_logs` 占位，不调用节点函数（也不查注册表）。`class_id` 查注册表只在 `enabled=True` 分支用于选节点函数（未注册回退到 `type`），与跳过禁用节点无关。条件边的 `condition` 从 `data.condition` 取，空字符串转 `None`。

## 参数与 mode

**Q：我改了算法参数为什么没生效？**
A：三级参数优先级 **前端 > 数据库 > 本地 JSON**，高优先级覆盖低优先级。前端 `data.params` 会盖掉你改的 DB / JSON。改完还要**发布流程 + reloadOne** 才生效。详见 [reference/03-key-concepts.md](../reference/03-key-concepts.md)。

**Q：两个 `mode` 有什么区别？**
A：节点级 `mode`（DAG 内 `ai_test` 节点）选算法模型；请求级 `mode`（`/detect/execute` 入参）是 `"0"`实际 / `"1"`全OK / `"2"`全NG 的调试开关。别混。

**Q：子流程（sub_step）怎么嵌套？**
A：`sub_step` 节点 one→one，内部独立执行一个 DAG 子流程。子流程懒加载——状态 draft 也会自动尝试加载。注意嵌套 fork 曾触发闸门死锁（git log `360a0b6` 已修），并发改动要小心 GPU 闸门。

## 软失败与异常

**Q：算法报 ERROR 但接口 status=success？**
A：`_detect_soft_failure` 只查顶层 `code` 会漏，必须**同时查嵌套 `ai_result.code`**（git log `45157a0`，CLAUDE.md 红线）。

**Q：响应里 `status` 有哪些值？**
A：`success` / `fail` / `algorithm_error` / `cached` / `skipped`；`source` 是 `executed` / `cached` / `skipped`。见 [reference/04-detect-engine.md](../reference/04-detect-engine.md)。

## 分支与提交

**Q：我的 fix 提到哪个分支？**
A：日常 feature/fix/chore 都在 `dev`。`master` 只接受 `git merge dev` + tag。算法 AI 改 `v2/models/**`/`v2/dag/**` 例外，可直接 master（见 [scenarios/03-add-algorithm-flow.md](../scenarios/03-add-algorithm-flow.md)）。

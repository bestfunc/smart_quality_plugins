# 场景 03 · 算法迭代与流程发布

> 受众：算法工程师 / 算法 AI。目标：换模型 / 调参数 → 热加载生效，不停机不重建镜像。

## 工作流：更新模型 / YAML（product_architecture_doc §3.3）

```
步骤 1 上传新模型制品  → Web 后台上传 model.pt + config.yaml 到 MinIO
步骤 2 创建新版本      → 后台创建模型制品新版本（V1.0 → V2.0，不覆盖历史）
步骤 3 关联到流程      → 编辑算法流程，选新版本制品
步骤 4 发布流程        → 调 reloadOne 加载（同步等待完成）

是否重打镜像？→ 不需要    是否重启服务？→ 不需要（热加载）    是否重部署？→ 不需要
```

> 多 worker 场景：master 接收 reloadOne 后自动级联加载到所有 worker。

## 细节：可改 / 不可改参数

| 参数 | 可否业务配置 |
|---|---|
| `n_fft` / `hop_length` / `num_mel_bins` / `target_length` / `label_config` | ❌ 必须与训练一致，改了推理全错 |
| `low_freq` / `high_freq` | ⚠️ 特定场景可调，需与算法工程师确认 |
| `split_duration`（切分时长） | ✅ |
| 评分 `dbs_rate` / `rms_rate` / `ffts_rate` / `sc_type` | ✅ 流程编辑器调 |

> **红线**：换模型必须同步更新配套 `config.yaml`；MinIO 模型用新建版本而非覆盖；直接改 DB 里的 flowData 会损坏 JSON，必须走 Web 后台。

## 示例：算法 AI 自助发布（CLAUDE.md 例外条款）

仅改动 `v2/models/**` / `v2/dag/**` 的算法迭代，算法 AI 可**直接在 master commit + tag + 部署**，跳过 dev：

1. 改 `v2/models/BF_*/` 或 `v2/dag/nodes/`
2. 本地 `pytest tests/test_dag_*.py` 通过
3. master commit + tag → scp `v2/` → 滚动重启 → 等 4/4 healthy
4. 调一次 `/detect/execute`（`mode=0`）验证结果

> 触碰框架 / compose / nginx / env 仍须走 dev → master 流程。完整自助流程见 `docs/algo_ai_deploy_guide.md`。

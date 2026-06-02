# 05 · 算法模型层（21+ BF_* 算法）

> 事实来源：`v2/models/`、`docs/product_architecture_doc.md` 附录 B、`docs/BF_*_design.md`。

## 定义：两类模型

| 类型 | 模型文件 | 配置文件 | 示例 |
|---|---|---|---|
| **PyTorch 深度学习模型** | `.pt` / `.safetensors` 权重 | `config.yaml`（预处理参数） | 音频分类、特征提取 |
| **规则算法模型** | 无模型文件 | JSON 参数（内置） | 评分、分贝、脉冲检测 |

模型由 `ModelAdmin` 动态加载（延迟加载 + 线程安全），`bf_manager.py` 是模型基类。

## 细节：算法清单

| 算法类型 | 编号 | 能力 | 需 GPU |
|---|---|---|---|
| 音频分类 | `BF_V14A` / `V15A` / `V16A` | ResNet + ViT 音频分类，识别声音类别 | ✅ 推荐 |
| 早期分类 | `BFPT_V12` | 早期分类网络 | ✅ |
| 特征提取 | `BF_FE_V1` / `V2` | ViT 提取音频特征向量，支持相似度匹配 | ✅ 推荐 |
| 异常检测 | `BF_FE_AD_V2` | 基于向量数据库（Qdrant）的异常声音检测 | ✅ 推荐 |
| 特征分类 | `BF_FE_CLASSES` | 特征分类 | ✅ |
| 多维评分 | `BF_SC_V1`~`V6` | dB / RMS / FFT 多维评分，支持百分制/百分比/计数 | ❌ |
| 分贝测量 | `BF_DB_V1` | 声压级测量，支持峰值/动态/相对模式 | ❌ |
| 线轮音/脉冲 | `BF_PS_V1` | FFT 峰值检测 + 频谱增强，检测电机/周期性异响 | ❌ |
| 感知评估 | `BF_CF_V1` | 频率域感知评估，判断频段内合格率 | ❌ |
| 仲裁判定 | `BF_JUDGE_V1` | 汇总多算法结果，加权仲裁输出最终判定 | ❌ |
| 通用基础 | `BF_COMMON`（bf_base/tiny/small） | 通用基础模型 | — |
| 演示 | `BF_DEMO_V1` | 演示用途 | ✅ |

> 评分算法 `BF_SC` 的可配业务参数：`dbs_rate`/`rms_rate`/`ffts_rate`（各维度权重）、`sc_type`（score 百分制 / percent 百分比 / count 计数）。参数定义见 `v2/models/BF_SC_V1/bf_manager_pt_sc.json`，其加载注入链路见 `docs/detect_rating_loading.md`。

## 示例：模型制品（pt + yaml 配套）

PyTorch 模型的权重与预处理配置必须配套，存放在 MinIO：

```
MOD-2026-0001/V1.0/
  ├── model.pt          ← PyTorch 权重
  └── config.yaml       ← 预处理配置（必须与模型匹配）
        preprocess_config:
          n_fft / hop_length / num_mel_bins / target_length / low_freq / high_freq
        label_config:
          分类标签定义
```

**预处理参数不可业务配置**（`n_fft` / `hop_length` / `num_mel_bins` / `target_length` / `label_config`）——必须与模型训练时一致，改动会导致推理结果完全错误。可业务配置的是 `split_duration`（切分时长）和评分权重类参数。

> **红线**：更换模型必须同时更新对应 `config.yaml`；删除/覆盖 MinIO 模型文件用**新建版本**而非覆盖。模型迭代流程见 [scenarios/03-add-algorithm-flow.md](../scenarios/03-add-algorithm-flow.md)。

## 外部依赖

| 系统 | 算法层用途 |
|---|---|
| **MinIO** | 存储模型制品（pt + yaml）+ 待检测音频 |
| **Qdrant** | 音频特征向量存储 + 最近邻检索，仅 `BF_FE` / `BF_FE_AD` 用 |
| **ClearML** | 模型训练追踪 + 制品管理（训练侧，推理服务只读取制品） |

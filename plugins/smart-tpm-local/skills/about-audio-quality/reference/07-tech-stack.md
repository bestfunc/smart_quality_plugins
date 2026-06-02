# 07 · 技术栈

> 事实来源：`requirements-v2-docker.txt`、`v2/requirements.txt`、`Dockerfile_v2.cuda`、`README.md`。

## 定义：运行时基线

| 项 | 版本 / 要求 |
|---|---|
| 语言 | Python 3.10.12（conda 环境 `venv`） |
| 推理框架 | PyTorch 2.3.x（CUDA 12.1） |
| 基础镜像 | PyTorch 2.3.1 + CUDA 12.1 |
| 数据库 | MySQL 8.0+ |

> 本机环境注意（CLAUDE.md 全局）：命令名是 `python` 不是 `python3`；含中文脚本写 `.py` 文件再跑，别用 heredoc。

## 细节：依赖分层（requirements-v2-docker.txt）

| 用途 | 关键依赖 |
|---|---|
| **Web / API** | `fastapi==0.115.0`、`uvicorn==0.31.0`、`python-multipart==0.0.12`、`gevent==24.11.1`、`websockets`、`watchdog==5.0.3` |
| **推理 / 模型** | `torch`(2.3, cu121)、`safetensors==0.4.5`、`torchinfo==1.8.0`、`einops==0.8.0` |
| **音频处理** | `librosa`、`soundfile`、`audiomentations`、`scipy==1.12.0`、`numpy==1.26.4` |
| **数据 / 科学** | `pandas==2.2.2`、`matplotlib==3.7.1`、`joblib`、`opencv-python` |
| **数据层** | `pymysql`、`DBUtils`、`minio`、`qdrant-client` |
| **MLOps** | `clearml==1.16.4` |
| **其它** | `pydantic`、`psutil`、`requests`、`PyYAML`、`python-dateutil`、`plotly`、`visualdl` |

## 示例：外部系统与连接要求

| 系统 | 协议 | 用途 | 凭据来源 |
|---|---|---|---|
| MySQL | PyMySQL / DBUtils | 流程配置、执行日志、API 密钥 | `db.yaml`（环境注入，不落盘到本百科） |
| MinIO | S3 兼容 HTTP | 模型制品 + 音频文件，支持 per-file 指定 `fileServer` | `db.yaml` 的 `minio.access_key`/`secret_key` |
| Qdrant | HTTP / gRPC | 特征向量存储与检索（仅特征类算法） | 环境配置 |
| ClearML | REST | 模型训练追踪与制品管理 | 环境配置 |

> 镜像构建命令、离线 pip 包管理见 `README.md` 的「常用运维命令」段。具体连接地址/端口/凭据见 `CLAUDE.md` 环境速查与 `db.yaml`——本百科一律用环境变量名占位，不写真实值。

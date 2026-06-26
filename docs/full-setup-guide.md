
# SuperMew 项目全流程部署指南

本文档基于开源项目 SuperMew 实操整理，覆盖从环境准备、依赖安装到服务启动的完整复现流程，记录部署过程中的关键步骤与注意事项。

## 一、前置环境准备

部署前请确保本机已安装以下工具：

- Git
- uv（Python 项目与依赖管理工具）
- Docker & Docker Compose
- Node.js 与 npm（用于前端代码编译）
- Python 3.12+

## 二、拉取项目代码

```bash
# 克隆项目仓库
git clone https://github.com/icey1287/SuperMew.git

# 进入项目根目录
cd SuperMew

# 同步最新代码（可选）
git pull
```

## 三、Python 环境与依赖安装

项目使用 uv 进行虚拟环境与依赖管理，基于 `pyproject.toml` 和 `uv.lock` 还原环境。

```bash
# 检查本机可用的 Python 版本，确认存在 Python 3.12
uv python list

# 创建项目虚拟环境并安装全部依赖
# 自动在项目根目录生成 .venv 虚拟环境
uv sync

# 激活环境
source .venv/bin/activate
```

## 四、环境变量配置

```bash
# 复制环境变量模板文件
cp .env.example .env
```

打开 `.env` 文件，按实际情况配置核心参数，以下为关键配置项说明：

### 模型 API 配置

```env
# ===== Model =====
ARK_API_KEY=你的模型API密钥
MODEL=glm-4.7
GRADE_MODEL=glm-4.5-air
FAST_MODEL=glm-4.5-air
BASE_URL=https://open.bigmodel.cn/api/paas/v4
```

> 可根据实际使用的模型服务商调整 `BASE_URL` 与对应模型名称。

## 五、Embedding 模型下载

项目使用 BAAI/bge-m3 作为向量嵌入模型，可通过 ModelScope 下载到本地。

```bash
# 安装 modelscope 工具，我没有下载到supermew环境，我怕弄坏环境
uv pip install -U modelscope

# 下载 bge-m3 模型到本地 models/bge-m3 目录
modelscope download --model BAAI/bge-m3 --local_dir ./models/bge-m3

# 测试模型加载是否正常（需先修改脚本内的模型路径）
python test_embedding.py
```

正常输出示例：向量维度为 1024。

## 六、基础服务容器化部署

项目依赖 Postgres、Redis、Milvus、MinIO 等基础服务，通过 Docker Compose 一键启动。

```bash
# 后台启动全部容器服务
docker compose up -d
```

启动完成后，查看所有服务运行状态：

```bash
docker compose ps
```

默认启动的服务清单：

| 服务名            | 镜像                        | 作用                  |
| ----------------- | --------------------------- | --------------------- |
| supermew-postgres | postgres:15                 | 关系型数据库          |
| supermew-redis    | redis:7-alpine              | 缓存与会话存储        |
| milvus-etcd       | quay.io/coreos/etcd:v3.5.18 | Milvus 元数据管理     |
| milvus-minio      | minio/minio                 | Milvus 对象存储       |
| milvus-standalone | milvusdb/milvus:v2.5.14     | 向量数据库            |
| milvus-attu       | zilliz/attu:v2.5.11         | Milvus 可视化管理界面 |

## 七、前端代码编译

首次运行或修改前端代码后，必须执行编译构建，生成静态资源供后端调用。

```bash
# 进入前端目录
cd frontend

# 安装前端依赖
npm install

# 编译构建静态包
npm run build
```

## 八、启动后端服务

回到项目根目录，启动后端 API 服务：

```bash
uv run uvicorn backend.app:app --host 0.0.0.0 --port 8000 --reload
```

浏览器访问：

- 前端页面：`http://127.0.0.1:8000/` （后端静态托管编译后的 `frontend/dist` 资源）
- API 文档：`http://127.0.0.1:8000/docs`

## 九、服务停止与清理

```bash
# 停止全部容器服务
docker compose down

# 彻底销毁容器与对应网络（清理环境时使用）
docker compose down --remove-orphans
```

## 参考资料

- 原项目仓库：https://github.com/icey1287/SuperMew
- 部署参考视频：https://www.bilibili.com/video/BV1zeQ7BkEa6/

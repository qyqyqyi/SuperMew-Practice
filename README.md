# SuperMew 实践：Agentic RAG 系统复现与评估体系搭建

## 项目简介

本项目基于 [SuperMew](https://github.com/icey1287/SuperMew) 开源 Agentic RAG 项目进行全流程复现与工程化实践，完整落地了从环境部署、系统运行到 LangSmith 效果评估的全链路。

项目以韩国签证知识库为测试场景，搭建了可量化的 RAG 离线评估流水线，同时定位并修复了部署与评估链路中的核心工程问题，落地了异步架构优化方案，并沉淀了完整的复现与排障文档。

## 核心工作与产出

### 1. 全流程复现落地

- 从零完成项目环境搭建、依赖适配、容器化基础服务部署，跑通「知识库上传 → Agent 推理检索 → 对话问答」完整功能链路
- 基于 LangSmith 搭建 RAG 效果评估体系，完成评测集构建、评估器配置、批量实验运行、效果对比全流程
- 基于韩国签证场景生成覆盖多难度的问答评测集，完成正确性、自定义指标的批量评估验证

### 2. 核心问题排查与修复

- **MinIO 容器 unhealthy 问题**：定位到精简镜像无 curl 工具导致健康检查失败的根因，替换为内置 `mc` 客户端检查方案，同时修复旧版密钥变量弃用警告，服务状态恢复正常
- **LangSmith 评估超时误判问题**：定位到后置大模型任务阻塞主链路、触发评估器超时重试导致 0.5 分异常的根因，从根源解决了正确答案被误判低分的问题

### 3. 架构性能优化

针对同步、异步两个对话入口分别做了**核心链路与后置任务解耦**优化，覆盖离线评估与在线服务两类场景，在保证功能完整与数据最终一致的前提下，全面降低响应耗时。

* **同步入口 `chat_with_agent`（评估脚本调用）** ：通过全局线程池 `ThreadPoolExecutor` 将会话标题生成、持久化笔记更新、历史消息落库等后置操作剥离为后台任务，核心回答生成后立即返回。既让评估耗时真实反映「检索 + 生成」的核心链路性能，也提升了批量评估实验的运行效率，从根源解决了评估超时导致的评分误判问题。
* **异步流式入口 `chat_with_agent_stream`（前端接口调用）** ：通过 `asyncio.create_task` 将收尾逻辑提交至事件循环后台执行，回答流与结束标记推送完成后立即释放 HTTP 连接；同步落库操作通过 `asyncio.to_thread` 放入线程池执行，避免阻塞事件循环。有效缩短前端加载等待时间，提升服务端连接资源利用率，保障高并发下的异步服务性能。

### 4. 工程化文档沉淀

- 输出完整的项目部署复现指南，覆盖全流程操作步骤与环境适配说明
- 整理核心问题的完整排查思路与解决方案，形成可复用的排障手册
- 梳理 LangSmith 评估体系搭建流程与生产级优化方向，形成标准化实践文档

## 技术栈

- **核心框架**：Python / LangChain / LangSmith
- **向量与存储**：Milvus / MinIO / PostgreSQL / Redis
- **部署与工具**：Docker / Docker Compose / uv / npm
- **模型能力**：智谱 AI 系列模型 / BAAI/bge-m3 Embedding

## 快速开始

1. 环境准备：安装 Docker、Python 3.12+、Node.js
2. 安装依赖：`uv sync`
3. 启动基础服务：`docker compose up -d`
4. 编译前端：`cd frontend && npm install && npm run build`
5. 启动后端：`uv run uvicorn backend.app:app --host 0.0.0.0 --port 8000 --reload`

> 完整详细步骤与注意事项见 [完整部署复现指南](./docs/full-setup-guide.md)

## 文档索引

| 文档                                                                   | 说明                                                 |
| ---------------------------------------------------------------------- | ---------------------------------------------------- |
| [完整部署复现指南](./docs/full-setup-guide.md)                            | 从零开始的全流程部署步骤、环境配置与操作命令         |
| [MinIO unhealthy 问题排查手册](./docs/troubleshooting-minio-unhealthy.md) | MinIO 健康检查异常的完整排查流程、根因分析与解决方案 |
| [LangSmith RAG 评估全流程指南](./docs/langsmith-rag-evaluation-guide.md)  | 评估体系搭建、问题排查、异步优化方案与生产级优化方向 |
| [原始项目官方文档](./ORIGINAL_README.md)                                  | 原 SuperMew 项目的完整官方说明文档                   |
| [开源协议](./LICENSE)                                                     | 项目沿用的开源许可证                                 |

## 项目目录结构

```
SuperMew/
├── backend/                # 后端核心代码
├── frontend/               # 前端代码
├── models/                 # 本地 Embedding 模型文件
├── docs/                   # 实践文档目录
│   ├── full-setup-guide.md
│   ├── troubleshooting-minio-unhealthy.md
│   └── langsmith-rag-evaluation-guide.md
├── ORIGINAL_README.md      # 原项目官方文档
├── LICENSE                 # 开源协议
├── docker-compose.yml      # 容器服务配置
├── pyproject.toml          # Python 依赖配置
└── README.md               # 项目说明
```

## 开源协议

本项目基于原 SuperMew 项目进行二次开发实践，沿用原项目的开源协议，完整协议内容见 [LICENSE](./LICENSE)。所有原始版权归原作者所有，本仓库仅用于学习与实践。

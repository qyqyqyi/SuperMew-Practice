LangSmith RAG 项目评估全流程实战指南

本文档基于 SuperMew 项目实践整理，覆盖 RAG 效果评估体系的完整搭建流程、核心问题排查方案与生产级优化方向，可用于复现 LangSmith 离线评估流水线。

## 一、评估体系概述

基于 LangSmith 搭建 RAG 系统离线评估流程，形成可量化的效果验证闭环，核心链路为：

1. 基于知识库内容自动生成标准化问答对作为评测集
2. 配置 LLM-as-Judge 正确性评估器与自定义规则校验
3. 通过 Python SDK 批量执行评估实验
4. 量化对比不同检索策略、提示词、模型版本的效果差异

相比人工评估更客观高效，支持迭代过程中的效果回归验证。

## 二、评测数据集构建

### 2.1 自动生成问答对

基于知识库文档片段，通过大模型批量生成覆盖不同难度的问答对，作为标准化评测样本。

生成提示词模板：

```plaintext
1. 问题要符合真实用户的提问习惯，不要太书面化
2. 答案必须完全来自文档片段，不能编造
3. 覆盖3种难度：简单事实题、总结归纳题、多细节推理题

文档片段：
{{document_chunk}}

请按JSON格式输出：
[{"question": "xxx", "answer": "xxx"}]
```

### 2.2 数据集导入 LangSmith

生成问答对后，整理为 CSV 或 JSONL 格式，直接上传至 LangSmith 平台的 **Datasets** 模块，完成评测集创建。

## 三、评估环境搭建与运行

### 3.1 环境变量配置

在项目 `.env` 或终端环境中配置 LangSmith 与模型服务的关键参数：

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_ENDPOINT=https://api.smith.langchain.com
export LANGSMITH_API_KEY=你的LangSmith API密钥
export OPENAI_API_KEY=你的模型服务API密钥
```

这些是临时变量

### 3.2 评估器创建

在 LangSmith 平台创建 Evaluator，选择 **LLM-as-Correctness** 评估器，配置对应模型与 API 密钥；可同时添加自定义规则评估器做基础校验。

本地核心依赖导入：

```python
from langsmith import Client, wrappers
from openevals.llm import create_llm_as_judge
from openevals.prompts import CORRECTNESS_PROMPT
from openai import OpenAI
```

### 3.3 执行评估实验

配置完成后，运行评估脚本批量调用目标接口，完成所有样本的测试。LangSmith 会自动统计正确率、耗时、Token 消耗等核心指标，生成完整的实验报告。

```plaintext
python test_embedding.py
```

## 四、核心问题排查：LLM-as-Judge 评分异常（0.5分误判）

### 4.1 问题现象

1. 明确回答正确的样本，correctness 评分出现异常 0.5 分，拉低整体平均分
2. 接口整体响应耗时偏高，普遍在 60s 以上，部分样本接近 100s
3. 单条样本的评估链路存在重试记录：首次判分 0 分、重试后判 1 分，最终取均值为 0.5

### 4.2 根因分析

chat_with_agent主链路同步执行了**生成会话标题、更新长久知识总结**两个非实时后置任务，这两个任务都需要额外调用大模型，导致整体响应时长大幅增加。

LangSmith 评估器存在等待超时机制：首次等待超时后直接判定为 0 分，触发重试后拿到完整答案判定为 1 分，最终取两次结果的均值得到 0.5 分。该问题属于超时导致的误判，并非答案本身质量问题。

### 4.3 排查过程

1. 对照单条样本的完整 Trace，确认模型最终输出内容完全匹配参考答案标准
2. 统计接口各阶段耗时，发现主回答生成仅占 30%-40%，大部分耗时来自后置的标题生成与笔记更新
3. 拆分代码执行链路，确认两个后置任务与主回答生成串行执行，阻塞了接口返回
4. 结合评估日志验证：首次评估超时返回 0 分，重试成功返回 1 分，与 0.5 分的结果完全对应

### 4.4 解决方案：后置任务异步剥离

使用 Python 内置线程池，将会话标题生成、持久化笔记更新、历史消息落库等非实时任务从主链路剥离，放到后台异步执行，主函数优先返回核心回答结果。

核心实现代码：

```python
from concurrent.futures import ThreadPoolExecutor

# 全局创建线程池，避免每次调用重复创建
_executor = ThreadPoolExecutor(max_workers=2)

def chat_with_agent(
    user_text: str,
    user_id: str = "default_user",
    session_id: str = "default_session",
):
    # 前置逻辑：加载历史上下文、构造输入
    messages, metadata = storage.load_with_meta(user_id, session_id)
    persistent_note = metadata.get("persistent_note", "")
    is_first_message = len(messages) == 0

    get_last_rag_context(clear=True)
    reset_knowledge_tool_calls()

    context_messages = _build_context_messages(messages, persistent_note, user_text)
    messages.append(HumanMessage(content=user_text))
    storage.save(user_id, session_id, messages)

    # 核心推理：执行 Agent 生成回答
    result = agent.invoke(
        {"messages": context_messages},
        config={"recursion_limit": 8},
    )

    # 解析回答内容
    response_content = ""
    if isinstance(result, dict):
        if "output" in result:
            response_content = result["output"]
        elif "messages" in result and result["messages"]:
            msg = result["messages"][-1]
            response_content = getattr(msg, "content", str(msg))
        else:
            response_content = str(result)
    elif hasattr(result, "content"):
        response_content = result.content
    else:
        response_content = str(result)

    messages.append(AIMessage(content=response_content))

    rag_context = get_last_rag_context(clear=True)
    rag_trace = rag_context.get("rag_trace") if rag_context else None

    # ========== 关键改动：后置任务异步执行，先返回结果 ==========
    def _post_process():
        """后台异步执行：生成标题、更新笔记、保存历史"""
        try:
            save_meta = dict(metadata)
            if is_first_message:
                save_meta["title"] = _generate_session_title_sync(user_text)
                save_meta["persistent_note"] = _update_persistent_note_sync(
                    persistent_note, user_text, response_content
                )
            extra_message_data = [None] * (len(messages) - 1) + [{"rag_trace": rag_trace}]
            storage.save(
                user_id, session_id, messages,
                metadata=save_meta,
                extra_message_data=extra_message_data,
            )
        except Exception as e:
            print(f"[后台任务异常] {e}")

    # 提交到线程池后台执行，不阻塞主流程返回
    _executor.submit(_post_process)

    # 直接返回核心结果，无需等待后置任务完成
    return {
        "response": response_content,
        "rag_trace": rag_trace,
    }
```

### 4.5 效果验证

1. 接口主链路耗时从平均 60s+ 降低至 40s 以内，消除了评估超时场景
2. LLM-as-Judge 评分恢复正常，正确答案全部判定为 1.0 分，无异常 0.5 分误判
3. 后置任务在后台正常执行，不影响用户体验与核心功能

## 五、生产级工程优化方向

基于 Demo 级评估体系，可从以下五个方向扩展为生产级方案，覆盖性能、成本、效率、可观测性全链路。

### 5.1 可观测性：动态采样策略

- **核心目标**：解决全量 Trace 拖慢主流程、浪费存储资源的问题，平衡可观测性与性能
- **落地方法**：
  1. 全局配置基础低采样率（如 20%），正常请求按比例上报
  2. 错误、超时、慢查询、空召回等异常场景强制 100% 全量采样并打标签
- **预期收益**：追踪数据量降低 70% 以上，主流程额外延迟控制在 5% 以内，核心异常零丢失

### 5.2 迭代效率：评估自动化 + CI 回归

- **核心目标**：解决手动评估效率低、代码迭代易出现效果退化的问题
- **落地方法**：
  1. 基于 pytest 封装评估脚本，设置基线指标断言（如正确率 ≥ 0.8），低于基线直接测试失败
  2. 拆分两套评测集：20 条冒烟集用于快速回归，全量集用于发版前深度评估
  3. 接入 CI 流水线，修改提示词、检索策略、模型版本时自动触发评估
- **预期收益**：策略迭代效率提升 80%，从根源避免优化过程中效果倒退

### 5.3 性能优化：前后置任务拆分

- **核心目标**：解决非核心任务拖慢整体响应、拉长用户等待时间的问题
- **落地方法**：
  1. 实时核心链路（检索 + 答案生成）优先执行，结果立刻返回
  2. 非实时后置任务（会话标题生成、持久化笔记更新、历史消息落库）全部剥离到后台
     - 同步函数使用 ThreadPoolExecutor 线程池执行
     - 异步流式函数使用 asyncio.create_task 执行
- **预期收益**：用户体感响应时间降低 40% 以上，HTTP 连接更快释放

### 5.4 成本优化：数据驱动模型选型

- **核心目标**：避免盲目使用大模型，在效果可接受范围内实现性价比最优
- **落地方法**：
  1. 统一封装模型调用层，自动统计每次调用的 Token 消耗与单次成本
  2. 固定同一套评测集，横向对比不同模型的「效果 + 耗时 + 成本」三个维度
  3. 分层选型：简单场景用轻量模型降本，复杂场景用大模型保效果
- **预期收益**：单查询平均算力成本降低 30%+，效果损失控制在可接受范围

### 5.5 问题排查：标准化 Bad Case 分析流程

- **核心目标**：解决问题定位难，无法区分是检索差还是生成差的痛点
- **落地方法（分层排查法）**：
  1. 先查检索层：确认召回片段中是否包含正确答案
     - 无 → 优化分块策略、嵌入模型、召回算法
     - 有 → 问题在生成层
  2. 再查生成层：确认模型是否正确使用了上下文信息
     - 没用到 → 优化提示词、更换可控性更强的模型
  3. 闭环归档：定位后的坏例加入评测集，后续优化自动验证修复效果
- **预期收益**：问题定位效率大幅提升，优化方向明确不盲目

## 参考资料

- 原项目仓库：https://github.com/icey1287/SuperMew
- 教程参考视频：https://www.bilibili.com/video/BV1mfAHzAEru/

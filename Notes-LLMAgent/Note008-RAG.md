## General RAG Architecture

**General RAG(Retrieval-Augmented Generation) Architecture**: Retrieval + Generation
<div align="center">
    <img src="https://github.com/bragai/bRAG-langchain/raw/main/assets/img/rag-architecture.png"/>
</div>

For detail: [bRAG](https://github.com/BragAI/bRAG-langchain/tree/main)



**概念解释**:

- **索引**

  - 通过 **Embedding** 将片段文本转换为向量
  - 将片段文本和片段向量存入 **向量数据库** 中
  - 向量模型排行榜：[Embedding Leaderboard](https://huggingface.co/spaces/mteb/leaderboard)

- **向量数据库**：用于存储和查询**向量**的数据库

- **召回**：搜索与用户问题相关的片段

  <img src="assets/image-20260202173739296.png" alt="image-20260202173739296" style="zoom:67%;" />

- **重排**：对召回的结果进行重新排序，挑选最终文本片段



**召回 VS 重排：**

- 召回

  ```python
  # 方法：向量相似度
  # 特点：1）成本低；2）耗时短；3）准去率较低
  # 适合场景：初步筛选
  ```

- 重拍

  ```python
  # 方法：cross-encoder
  # 特点：1）成本高；2）耗时长；3）准去率较高
  # 适合场景：精细挑选
  ```





## RAG Workflow

<img src="assets/image-20260202174623285.png" alt="image-20260202174623285" style="zoom:67%;" />





## LangChain RAG Workflow

[LangChain RAG Documentation](https://docs.langchain.com/oss/python/langchain/rag)


### Indexing

Indexing commonly works as follows:

1. **Load**: First we need to load our data. This is done with [Document Loaders](https://docs.langchain.com/oss/python/langchain/retrieval#document_loaders).
2. **Split**: [Text splitters](https://docs.langchain.com/oss/python/langchain/retrieval#text_splitters) break large `Documents` into smaller chunks. This is useful both for indexing data and passing it into a model, as large chunks are harder to search over and won’t fit in a model’s finite context window.
3. **Store**: We need somewhere to store and index our splits, so that they can be searched over later. This is often done using a [VectorStore](https://docs.langchain.com/oss/python/langchain/retrieval#vectorstores) and [Embeddings](https://docs.langchain.com/oss/python/langchain/retrieval#embedding_models) model.

<div align="center">
    <img src="https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/rag_indexing.png?w=840&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=1838328a870c7353c42bf1cc2290a779"/>
</div>




### Retrieval and generation

RAG applications commonly work as follows:

1. **Retrieve**: Given a user input, relevant splits are retrieved from storage using a [Retriever](https://docs.langchain.com/oss/python/langchain/retrieval#retrievers).
2. **Generate**: A [model](https://docs.langchain.com/oss/python/langchain/models) produces an answer using a prompt that includes both the question with the retrieved data

<div align="center">
    <img src="https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/rag_retrieval_generation.png?w=840&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=67fe2302e241fc24238a5df1cf56573d"/>
</div>







## RAG Evaluation

### RAG Evaluation Guide

[RAG Evaluation Guide](https://deepeval.com/guides/guides-rag-evaluation#evaluating-retrieval)

<div align="center">
    <img src="https://d2lsxfc3p6r9rv.cloudfront.net/rag-pipeline.svg" width="500" />
</div>


`deepeval` offers three LLM evaluation metrics to evaluate retrievals:
- [`ContextualPrecisionMetric`](https://deepeval.com/docs/metrics-contextual-precision): evaluates whether the **reranker** in your retriever ranks more relevant nodes in your retrieval context higher than irrelevant ones.
- [`ContextualRecallMetric`](https://deepeval.com/docs/metrics-contextual-recall): evaluates whether the **embedding model** in your retriever is able to accurately capture and retrieve relevant information based on the context of the input.
- [`ContextualRelevancyMetric`](https://deepeval.com/docs/metrics-contextual-relevancy): evaluates whether the **text chunk size** and **top-K** of your retriever is able to retrieve information without much irrelevancies.


`deepeval` offers two LLM evaluation metrics to evaluate **generic** generations:

- [`AnswerRelevancyMetric`](https://deepeval.com/docs/metrics-answer-relevancy): evaluates whether the **prompt template** in your generator is able to instruct your LLM to output relevant and helpful outputs based on the `retrieval_context`.
- [`FaithfulnessMetric`](https://deepeval.com/docs/metrics-faithfulness): evaluates whether the **LLM** used in your generator can output information that does not hallucinate **AND** contradict any factual information presented in the `retrieval_context`.




### RAG Eval

使用DeepEval框架对RAG系统进行评测，包含以下指标：

- **Faithfulness**: 评估答案是否忠实于检索到的上下文
- **Correctness**: 使用G-Eval评估答案的正确性
- **BERT Similarity**: 基于BERT模型计算语义相似度



**评测流程**：
1. 加载预定义的问答对数据集
2. 调用AgentCore Runtime获取系统回答
3. 使用多个评测指标进行评分
4. 生成评测报告


**具体步骤**：
1. 安装依赖：
   ```bash
   pip install -r ./evaluation/requirements.txt
   ```
2. 执行评测：
   ```bash
   python ./evaluation/using_deepeval.py
   ```







## RAG Projects

**开源RAG项目**：

- [使用Python简单实现RAG](https://github.com/MarkTechStation/VideoCode/tree/main)
- [bRAG](https://github.com/BragAI/bRAG-langchain/tree/main): This repository contains a comprehensive exploration of Retrieval-Augmented Generation (RAG) for various applications. Each notebook provides a detailed, hands-on guide to setting up and experimenting with RAG from an introductory level to advanced implementations, including multi-querying and custom RAG builds.
- [RAGFlow](https://github.com/infiniflow/ragflow): RAGFlow is a leading open-source Retrieval-Augmented Generation (RAG) engine that fuses cutting-edge RAG with Agent capabilities to create a superior context layer for LLMs










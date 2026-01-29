# AI 中台中 RAG 设计方案v1-0

## 一、实现思考

> 核心思考：针对复杂多变的源数据，如何构建企业级 RAG , 问答效果好。

### 问题

1. 针对不同的数据文件，如（pdf, ppt, csv, excel 等）主流的数据解析流程是如何做的，我们的场景（数据）有什么特点？

2. 如何评价我们问答效果的好坏，评价指标如何定？不同的文档不同的指标吗？

3. 测试集合如何构建？利用大模型自动标注相关文档，如何构建自动的 pipeline ?

4. 是针对不同的文档构建不同的 pipeline 吗？

工作思路：第一步应该调研清楚当前业界的 RAG 方案，例如 RAGflow 的实现思路，尤其在数据解析这块的内容，当前的流程是如何的？在同样的标准下，现有的效果能达到什么样？

## 二、工作任务

### 2.1 RAGflow 相关接口调研

1. 利用 docker 部署 RAGflow，了解 ragflow 的基本工作流程。
2. 利用源码部署 RAGflow，方便实现二次开发和调研、评测。
3. 了解清楚 ragflow 的功能边界，使用方法，输出相关**功能文档**，方便下一步的任务
4. 了解 ragflow 内部的评测流程和接口的调用。 

### 2.2 RAGflow文档解析流程相关调研

- 如何分 chunks

- DeepDoc、MinerU 等有什么区别？什么类型的文档应该用什么样的解析方法

- naive、book、tabled 等又是针对什么情况的数据的，和上述的有啥区别？

### 2.3 评测集的构建

```json
{
"question": "月卡套餐都有什么",
"reference_answer": "有lite,boost,power等类型",
"relevant_doc_ids": ["doc_001"],
"relevant_chunk_ids": ["chunk_123"],
"metadata": {"doc_type": "table","difficulty": "medium","category": "产品说明"}
}
```

- 这里应该针对不同的文档构建不同的指标呢？还是怎么办？

- 如何构建 pipeline 实现测试集的构建，事实上主要是标注  relevant_chunk_ids，这样才能计算客观指标。

## 三、任务进度列表

### 3.1 ragflow 功能的初步

# WeKnora vs RAGFlow：Agentic RAG 设计对比分析

## 🎯 核心区别（一句话总结）

**最核心的区别**：

- **WeKnora**：**工具调用模式（Tool Calling Pattern）** - LLM 通过 Function Calling **直接调用工具**

- **RAGFlow**：**推理链提取模式（Reasoning Chain Extraction Pattern）** - LLM 生成推理文本，系统**从中提取查询**

> ⚠️ **重要澄清**：两者都使用了提示词工程（Prompt Engineering），但核心区别在于**工具调用的方式**和**执行流程的控制机制**，而非"提示词工程"本身。

## 📋 目录

- [核心区别详解](#核心区别详解)

- [一、核心架构差异](#一核心架构差异)

- [二、详细功能对比](#二详细功能对比)

- [三、关键设计位置](#三关键设计位置)

- [四、设计理念差异](#四设计理念差异)

- [五、主要区别总结](#五主要区别总结)

- [六、适用场景](#六适用场景)

- [七、代码示例对比](#七代码示例对比)

---

## 核心区别详解

### 两种执行模式的根本差异

#### WeKnora：工具调用模式（ReAct Pattern）

**核心机制**：**Function Calling（函数调用）**

```
LLM 思考 → 决定调用工具 → 直接调用工具（结构化） → 获取结果 → 继续思考
```

**特点**：

- LLM **直接调用工具**，工具调用是**结构化数据**（JSON）

- 工具调用和结果都是**标准格式**（OpenAI Function Calling）

- LLM **自主决定**调用哪个工具、何时停止

- 这是标准的 **ReAct（Reasoning + Acting）模式**

**示例**：

```json
// LLM 返回结构化工具调用

{

  "tool_calls": [{

    "id": "call_123",

    "function": {

      "name": "knowledge_search",

      "arguments": "{\"queries\": [\"什么是 RAG？\"]}"

    }

  }]

}
```

#### RAGFlow：推理链提取模式（Reasoning Chain Extraction）

**核心机制**：**标记提取（Tag Extraction）**

```
LLM 生成推理文本 → 系统提取查询标记 → 执行检索 → LLM 提取信息 → 继续推理
```

**特点**：

- LLM **生成自然语言推理文本**，包含特殊标记

- 系统**解析文本**，提取查询标记（`<|begin_search_query|>`）

- 查询提取是**文本解析过程**，不是结构化调用

- 这是**显式推理链模式**，强调推理过程的可视化

**示例**：

```python
# LLM 生成推理文本

reasoning = """

我需要查找 RAG 的信息。

<|begin_search_query|>什么是 RAG？<|end_search_query|>

"""



# 系统提取查询（文本解析）

queries = extract_between(reasoning, BEGIN_SEARCH_QUERY, END_SEARCH_QUERY)
```

### 为什么不是"提示词工程"的区别？

**两者都使用了提示词工程**：

- WeKnora：使用系统提示词指导 Agent 如何使用工具

- RAGFlow：使用推理提示词指导 LLM 如何生成推理链

**真正的区别在于**：

| 维度 | WeKnora | RAGFlow |

|------|---------|---------|

| **工具调用方式** | Function Calling（结构化） | 标记提取（文本解析） |

| **执行控制** | LLM 自主决策（finish_reason） | 系统解析推理文本 |

| **交互格式** | 结构化 JSON | 自然语言 + 标记 |

| **工具扩展** | 标准 Function Calling 接口 | 需要修改提取逻辑 |

### 类比理解

**WeKnora** 就像：

- 🎯 **API 调用**：直接调用函数，参数是结构化数据

- 类似：`knowledge_search(queries=["什么是 RAG？"])`

**RAGFlow** 就像：

- 📝 **文本解析**：从自然语言文本中提取信息

- 类似：从"我需要查找 `<|query|>什么是 RAG？</query|>`"中提取查询

---

## 一、核心架构差异

### WeKnora：ReAct Agent 模式

**位置**：`/home/zhaozhiming/AI/WeKnora/internal/agent/engine.go`

**架构特点**：

- 基于 **ReAct（Reasoning + Acting）循环**

- 使用标准的 **Function Calling** 机制

- 支持多轮迭代执行（最多 `MaxIterations` 轮）

- 每轮执行流程：**Think → Act（工具调用）→ Observe → 继续或结束**

**核心流程**：

```
用户查询 → 初始化 Agent Engine → ReAct 循环

    ↓

Round 1: Think → 决定调用工具 → Act（执行工具）→ Observe（获取结果）

    ↓

Round 2: Think → 决定调用工具 → Act → Observe

    ↓

...（最多 MaxIterations 轮）

    ↓

最终答案生成
```

### RAGFlow：Deep Research 模式

**位置**：`/home/zhaozhiming/AI/ragflow/agentic_reasoning/deep_research.py`

**架构特点**：

- 基于 **显式推理链的搜索循环**

- 使用 **特殊标记提取** 机制（`<|begin_search_query|>`）

- 固定搜索循环（最多 6 轮）

- 每轮执行流程：**生成推理 → 提取查询 → 检索 → 提取相关信息**

**核心流程**：

```
用户查询 → 初始化 DeepResearcher → 推理循环

    ↓

Step 1: 生成推理（包含查询标记）→ 提取查询 → 检索信息 → 提取相关信息

    ↓

Step 2: 生成推理 → 提取查询 → 检索信息 → 提取相关信息

    ↓

...（最多 6 轮）

    ↓

生成最终答案
```

---

## 二、详细功能对比

| 特性 | WeKnora | RAGFlow |

|------|---------|---------|

| **执行模式** | ReAct 循环（Think-Act-Observe） | 推理链循环（Reasoning-Search-Extract） |

| **工具调用方式** | Function Calling（标准 OpenAI 格式） | 特殊标记提取（`<\|begin_search_query\|>`） |

| **查询生成** | LLM 自主决定调用哪个工具 | LLM 在推理中生成查询，系统提取 |

| **迭代控制** | 动态：LLM 决定是否继续（`finish_reason="stop"`） | 固定：最多 6 轮，或 LLM 不再生成查询 |

| **工具类型** | 多样化：knowledge_search, grep_chunks, web_search, todo_write, list_knowledge_chunks 等 | 单一：主要是检索工具（KB + Web + KG） |

| **反思机制** | 可选：`ReflectionEnabled` 配置 | 内置：每轮检索后提取相关信息 |

| **最终答案生成** | 如果达到最大轮次，单独生成最终答案 | 推理链完成后直接生成答案 |

| **上下文管理** | ContextManager 持久化对话历史 | 消息历史维护，智能截断 |

| **流式输出** | 支持思考过程、工具调用、结果的流式输出 | 支持推理过程的流式输出 |

| **事件系统** | EventBus 事件驱动架构 | 直接 yield 返回结果 |

| **多知识库支持** | ✅ 支持跨知识库检索 | ✅ 支持多知识库检索 |

| **Web 搜索** | ✅ 支持（DuckDuckGo/Google） | ✅ 支持（Tavily） |

| **知识图谱** | ✅ 支持（Neo4j） | ✅ 支持 |

| **MCP 工具集成** | ✅ 支持 MCP 工具扩展 | ❌ 不支持 |

| **任务规划** | ✅ 支持 todo_write 工具 | ❌ 不支持 |

---

## 三、关键设计位置

### WeKnora 的核心文件

#### 1. Agent 引擎

**文件**：`internal/agent/engine.go`

**关键方法**：

- `Execute()` (第 77-155 行)：Agent 执行入口

- `executeLoop()` (第 157-494 行)：主 ReAct 循环

- `streamThinkingToEventBus()` (第 683-761 行)：思考过程流式输出

- `streamFinalAnswerToEventBus()` (第 763-868 行)：最终答案生成

- `appendToolResults()` (第 514-585 行)：将工具结果添加到消息历史

**关键代码片段**：

```go
// 主循环结构

for state.CurrentRound < e.config.MaxIterations {

    // 1. Think: 调用 LLM 思考并决定工具调用

    response, err := e.streamThinkingToEventBus(ctx, messages, tools, ...)

    // 2. 检查是否完成

    if response.FinishReason == "stop" && len(response.ToolCalls) == 0 {

        state.FinalAnswer = response.Content

        state.IsComplete = true

        break

    }

    // 3. Act: 执行工具调用

    for _, tc := range response.ToolCalls {

        result, err := e.toolRegistry.ExecuteTool(ctx, tc.Function.Name, ...)

        // 存储工具调用结果

    }

    // 4. Observe: 将工具结果添加到消息历史

    messages = e.appendToolResults(ctx, messages, step)

    state.CurrentRound++

}
```

#### 2. 系统提示词

**文件**：`internal/agent/prompts.go`

**关键提示词**：

- `ProgressiveRAGSystemPrompt` (第 355-432 行)：Progressive RAG 系统提示词

- `PureAgentSystemPrompt` (第 332-353 行)：纯 Agent 模式提示词

**核心设计理念**：

- **Evidence-First**：必须基于检索到的证据回答

- **Deep Read 强制要求**：检索后必须调用 `list_knowledge_chunks` 读取完整内容

- **KB First, Web Second**：优先使用知识库，再使用 Web 搜索

#### 3. 工具实现

**文件位置**：`internal/agent/tools/`

**主要工具**：

- `knowledge_search.go`：知识库语义搜索（支持向量检索、重排序、MMR）

- `grep_chunks.go`：关键词搜索

- `list_knowledge_chunks.go`：深度读取工具（强制要求）

- `web_search.go`：Web 搜索（支持 RAG 压缩）

- `web_fetch.go`：网页内容抓取

- `todo_write.go`：任务规划工具

- `query_knowledge_graph.go`：知识图谱查询

- `mcp_tool.go`：MCP 工具集成

### RAGFlow 的核心文件

#### 1. Deep Research 实现

**文件**：`agentic_reasoning/deep_research.py`

**关键方法**：

- `thinking()` (第 168-238 行)：主推理循环

- `_generate_reasoning()` (第 54-69 行)：生成推理步骤

- `_extract_search_queries()` (第 71-77 行)：从推理中提取搜索查询

- `_retrieve_information()` (第 99-127 行)：多源检索（KB + Web + KG）

- `_extract_relevant_info()` (第 147-166 行)：从检索结果中提取相关信息

- `_truncate_previous_reasoning()` (第 79-97 行)：智能截断历史推理

**关键代码片段**：

```python
# 主循环结构

for step_index in range(MAX_SEARCH_LIMIT + 1):

    # 1. 生成推理（包含查询标记）

    async for ans in self._generate_reasoning(msg_history):

        query_think = ans

    # 2. 提取搜索查询

    queries = self._extract_search_queries(query_think, question, step_index)

    if not queries and step_index > 0:

        break

    # 3. 处理每个查询

    for search_query in queries:

        # 4. 检索信息

        kbinfos = self._retrieve_information(search_query)

        # 5. 提取相关信息

        async for ans in self._extract_relevant_info(...):

            summary_think = ans

        # 6. 更新消息历史

        msg_history.append({"role": "user", "content": ...})
```

#### 2. 提示词定义

**文件**：`agentic_reasoning/prompts.py`

**关键提示词**：

- `REASON_PROMPT` (第 23-103 行)：推理提示词

  - 要求 LLM 将问题分解为可验证的步骤

  - 使用特殊标记 `<|begin_search_query|>` 和 `<|end_search_query|>` 标记查询

  - 提供多跳问题的示例

- `RELEVANT_EXTRACTION_PROMPT` (第 105-147 行)：信息提取提示词

  - 从检索结果中提取最相关的信息

  - 要求简洁、准确

**特殊标记**：

```python
BEGIN_SEARCH_QUERY = "<|begin_search_query|>"

END_SEARCH_QUERY = "<|end_search_query|>"

BEGIN_SEARCH_RESULT = "<|begin_search_result|>"

END_SEARCH_RESULT = "<|end_search_result|>"

MAX_SEARCH_LIMIT = 6
```

#### 3. 调用入口

**文件**：`api/db/services/dialog_service.py`

**关键位置**：第 373-395 行

```python
if prompt_config.get("reasoning", False):

    reasoner = DeepResearcher(

        chat_mdl,

        prompt_config,

        partial(retriever.retrieval, ...),

    )

    async for think in reasoner.thinking(kbinfos, attachments_ + " ".join(questions)):

        if isinstance(think, str):

            thought = think

        elif stream:

            yield think
```

---

## 四、设计理念差异

### WeKnora：工具导向的 Agent

**设计哲学**：

- **工具即能力**：通过丰富的工具集扩展 Agent 能力

- **自主决策**：LLM 自主决定调用哪些工具、何时停止

- **深度阅读**：强制要求深度阅读文档内容，不依赖检索摘要

- **任务规划**：支持多步骤任务规划（todo_write）

**执行流程示例**：

```go
// WeKnora 的执行流程

for round < MaxIterations {

    // 1. Think: LLM 思考并决定调用哪些工具

    response := LLM.Think(messages, tools)

    // 2. Act: 执行工具调用

    if response.ToolCalls != nil {

        for toolCall := range response.ToolCalls {

            result := ExecuteTool(toolCall)

            // 如果是检索工具，强制调用 list_knowledge_chunks

            if toolCall.Name == "knowledge_search" {

                chunks := list_knowledge_chunks(result.chunk_ids)

            }

        }

    }

    // 3. Observe: 将工具结果加入上下文

    messages = append(messages, toolResults)

    // 4. 判断：LLM 决定是否继续

    if response.FinishReason == "stop" {

        break

    }

}
```

**特点**：

- ✅ 工具丰富（knowledge_search, grep_chunks, list_knowledge_chunks, web_search, todo_write 等）

- ✅ 强调 "Deep Read"：检索后必须调用 `list_knowledge_chunks` 读取完整内容

- ✅ 支持 MCP 工具扩展

- ✅ 支持反思机制（ReflectionEnabled）

- ✅ 事件驱动架构，便于监控和调试

### RAGFlow：推理链导向的搜索

**设计哲学**：

- **显式推理**：要求 LLM 展示完整的思考过程

- **查询提取**：从推理文本中提取查询，而非直接工具调用

- **信息提取**：每轮检索后 LLM 提取相关信息

- **简化工具集**：聚焦检索相关工具

**执行流程示例**：

```python
# RAGFlow 的执行流程

for step_index in range(MAX_SEARCH_LIMIT):

    # 1. Generate Reasoning: LLM 生成推理步骤（包含查询标记）

    reasoning = LLM.GenerateReasoning(msg_history)

    # 推理文本示例：

    # "我需要查找 X 的信息。

    #  <|begin_search_query|>什么是 X？<|end_search_query|>"

    # 2. Extract Queries: 从推理中提取搜索查询

    queries = extract_between(reasoning, BEGIN_SEARCH_QUERY, END_SEARCH_QUERY)

    # 3. Retrieve: 执行检索（KB + Web + KG）

    kbinfos = retrieve_information(queries)

    # 4. Extract Info: LLM 从检索结果中提取相关信息

    relevant_info = LLM.ExtractRelevantInfo(kbinfos, queries)

    # 5. 判断：如果没有新查询则结束

    if not queries:

        break
```

**特点**：

- ✅ 显式推理链：LLM 必须展示思考过程

- ✅ 查询提取：从推理文本中提取查询（而非直接工具调用）

- ✅ 信息提取：每轮检索后 LLM 提取相关信息

- ✅ 简化工具集：主要是检索相关

- ✅ 推理过程可视化：用户可以看到完整的推理链

---

## 五、主要区别总结

### 1. 工具调用方式

**WeKnora**：

- 使用标准的 Function Calling 机制

- LLM 直接调用工具，工具返回结构化结果

- 工具调用格式遵循 OpenAI 标准

**RAGFlow**：

- 使用特殊标记提取机制

- LLM 在推理文本中嵌入查询标记，系统提取

- 更灵活但需要额外的解析步骤

### 2. 迭代控制

**WeKnora**：

- LLM 动态决定是否继续（`finish_reason="stop"`）

- 更灵活，可以根据任务复杂度调整迭代次数

- 支持提前终止

**RAGFlow**：

- 固定上限（6 轮）或 LLM 不再生成查询

- 更可预测，但可能在某些复杂任务上不够灵活

### 3. 工具丰富度

**WeKnora**：

- 工具丰富：检索、计划、深度读取、Web 搜索、知识图谱等

- 支持 MCP 工具扩展

- 工具之间可以组合使用

**RAGFlow**：

- 聚焦检索相关工具

- 工具集相对简单，但专注于检索增强

### 4. 深度阅读策略

**WeKnora**：

- **强制要求**：检索后必须调用 `list_knowledge_chunks` 进行深度阅读

- 系统提示词明确要求："MANDATORY Deep Read"

- 确保 Agent 基于完整内容而非摘要回答

**RAGFlow**：

- 检索后通过 LLM 提取相关信息

- 依赖 LLM 的能力从检索结果中提取关键信息

- 可能在某些情况下遗漏细节

### 5. 最终答案生成

**WeKnora**：

- 如果达到最大轮次，单独生成最终答案

- 会汇总所有工具调用的结果

- 支持更复杂的答案合成

**RAGFlow**：

- 推理链完成后直接生成答案

- 答案生成与推理过程集成

- 更简洁但可能缺少最终汇总步骤

### 6. 事件系统

**WeKnora**：

- EventBus 事件驱动架构

- 支持思考过程、工具调用、结果的流式输出

- 便于监控、调试和扩展

**RAGFlow**：

- 直接 yield 返回结果

- 更简单的实现，但扩展性相对较弱

---

## 六、适用场景

### WeKnora 适合的场景

1. **复杂任务规划**

   - 需要多步骤任务分解

   - 需要任务状态跟踪（todo_write）

   - 需要动态调整执行计划

2. **深度文档理解**

   - 需要完整阅读文档内容

   - 需要理解文档结构和关系

   - 需要跨文档信息整合

3. **工具扩展需求**

   - 需要集成外部工具（MCP）

   - 需要自定义工具

   - 需要工具组合使用

4. **企业级应用**

   - 需要完整的监控和日志

   - 需要事件驱动的架构

   - 需要灵活的配置和扩展

### RAGFlow 适合的场景

1. **显式推理需求**

   - 需要展示完整的推理过程

   - 需要推理链可视化

   - 需要可解释的 AI

2. **简单检索增强**

   - 主要是检索增强生成

   - 不需要复杂的工具编排

   - 聚焦于信息检索和提取

3. **快速原型开发**

   - 实现相对简单

   - 配置较少

   - 易于理解和调试

4. **教育演示**

   - 推理过程清晰可见

   - 适合教学和演示

   - 便于理解 RAG 工作原理

---

## 七、代码示例对比

### WeKnora：工具调用示例

```go
// WeKnora 中 LLM 的响应格式

{

    "content": "我需要查找关于 RAG 的信息。",

    "tool_calls": [

        {

            "id": "call_123",

            "type": "function",

            "function": {

                "name": "knowledge_search",

                "arguments": "{\"queries\": [\"什么是 RAG？\", \"RAG 的工作原理\"]}"

            }

        }

    ],

    "finish_reason": null

}



// 工具执行后，结果添加到消息历史

{

    "role": "tool",

    "tool_call_id": "call_123",

    "name": "knowledge_search",

    "content": "=== Search Results ===\nFound 5 relevant results..."

}
```

### RAGFlow：查询提取示例

```python
# RAGFlow 中 LLM 的推理文本

reasoning_text = """

我需要查找关于 RAG 的信息。

首先，我需要了解什么是 RAG。

<|begin_search_query|>什么是 RAG？<|end_search_query|>

然后，我需要了解 RAG 的工作原理。

<|begin_search_query|>RAG 的工作原理是什么？<|end_search_query|>

"""



# 系统提取查询

queries = extract_between(reasoning_text,

                         BEGIN_SEARCH_QUERY,

                         END_SEARCH_QUERY)

# queries = ["什么是 RAG？", "RAG 的工作原理是什么？"]



# 执行检索

kbinfos = retrieve_information(queries)



# LLM 提取相关信息

relevant_info = LLM.ExtractRelevantInfo(kbinfos, queries)
```

---

## 八、总结

### 核心差异

| 维度 | WeKnora | RAGFlow |

|------|---------|---------|

| **设计理念** | 工具导向的 Agent | 推理链导向的搜索 |

| **工具调用** | Function Calling | 标记提取 |

| **迭代控制** | 动态（LLM 决定） | 固定（最多 6 轮） |

| **工具丰富度** | 高（多种工具） | 中（聚焦检索） |

| **深度阅读** | 强制要求 | 依赖 LLM 提取 |

| **扩展性** | 高（MCP 支持） | 中（相对简单） |

| **可解释性** | 中（工具调用可见） | 高（推理链可见） |

### 选择建议

**选择 WeKnora 如果**：

- 需要复杂的工具编排

- 需要深度文档理解

- 需要任务规划能力

- 需要工具扩展（MCP）

- 需要企业级监控和日志

**选择 RAGFlow 如果**：

- 需要显式推理过程

- 主要是检索增强生成

- 需要推理链可视化

- 实现相对简单

- 适合教育和演示

### 未来发展方向

**WeKnora**：

- 继续扩展工具生态

- 增强任务规划能力

- 优化深度阅读策略

- 提升工具组合能力

**RAGFlow**：

- 优化推理链生成

- 增强信息提取能力

- 支持更多检索策略

- 提升推理过程可视化

---

## 参考资料

- **WeKnora 项目**：`/home/zhaozhiming/AI/WeKnora`

- **RAGFlow 项目**：`/home/zhaozhiming/AI/ragflow`

- **WeKnora Agent Engine**：`internal/agent/engine.go`

- **RAGFlow Deep Research**：`agentic_reasoning/deep_research.py`

---

*文档生成时间：2026年*

*对比版本：WeKnora v0.2.10, RAGFlow latest*

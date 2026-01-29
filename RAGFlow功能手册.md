# RAGFlow 功能手册 v1.1

> **定位**：RAGFlow 各功能详解、源码路径、操作参考

---

## 目录

1. [核心功能概览](#一核心功能概览)
2. [解析器详解](#二解析器详解)
3. [评测系统](#三评测系统)
4. [知识库与元数据管理](#四知识库与元数据管理)
5. [完整源码路径索引](#五完整源码路径索引)
6. [参考文档索引](#六参考文档索引)

---

## 一、核心功能概览

### 1.1 系统架构

RAGFlow 采用微服务架构，支持灵活的组件组合：

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端层 (React + UmiJS)                     │
├─────────────────────────────────────────────────────────────────┤
│                        API 层 (Flask 蓝图)                        │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐  │
│  │kb_app   │dialog   │document │evaluation│canvas   │file_    │  │
│  │         │_app     │_app     │_app     │_app     │app      │  │
│  └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        服务层 (Business Logic)                   │
├─────────────────────────────────────────────────────────────────┤
│                        数据层 (ORM Models)                       │
├─────────────────────────────────────────────────────────────────┤
│                        核心层 (RAG Pipeline)                     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 技术栈

| 组件   | 技术选型                                              |
| ---- | ------------------------------------------------- |
| 后端框架 | Flask                                             |
| 前端框架 | React + UmiJS                                     |
| 数据库  | MySQL                                             |
| 向量检索 | Elasticsearch / Infinity / OpenSearch / OceanBase |
| 缓存   | Redis                                             |
| 文件存储 | MinIO                                             |
| 容器化  | Docker + Docker Compose                           |

### 1.3 核心模块

| 模块        | 文件路径                         | 功能说明          |
| --------- | ---------------------------- | ------------- |
| 知识库管理     | `api/apps/kb_app.py`         | 知识库 CRUD、权限管理 |
| 对话系统      | `api/apps/dialog_app.py`     | 对话创建、知识库关联    |
| 文档处理      | `api/apps/document_app.py`   | 文档上传、解析、管理    |
| 评估系统      | `api/apps/evaluation_app.py` | 测试集、评估运行、结果分析 |
| Agent 工作流 | `api/apps/canvas_app.py`     | 工作流编排、组件管理    |

---

## 二、解析器详解

### 2.1 三种解析器对比表

| 特性           | General            | Book              | Table              |
| ------------ | ------------------ | ----------------- | ------------------ |
| **文件路径**     | `rag/app/naive.py` | `rag/app/book.py` | `rag/app/table.py` |
| **适用场景**     | 通用文档（PDF/DOCX/TXT） | 长文档、书籍、手册         | 表格数据（Excel/CSV）    |
| **Chunk 大小** | 512 tokens         | 256 tokens        | 每行一个 chunk         |
| **特殊处理**     | 支持多种版面识别           | 自动处理目录、层级结构       | 数据类型推断             |

### 2.2 版面识别方案对比

| 方案         | 速度  | 质量  | 适用场景 |
| ---------- | --- | --- | ---- |
| DeepDOC    | 中   | 高   | 通用文档 |
| MinerU     | 慢   | 最高  | 复杂布局 |
| Docling    | 中   | 高   | 学术论文 |
| Plain Text | 快   | 基础  | 简单文档 |

### 2.3 快速参考：文档类型推荐配置

| 文档类型         | 推荐解析器        | 推荐版面识别         | chunk_token_num |
| ------------ | ------------ | -------------- | --------------- |
| 普通 PDF/DOCX  | naive        | DeepDOC        | 512             |
| 学术论文         | paper        | MinerU/Docling | 512             |
| 书籍/长文档       | book         | DeepDOC        | 256             |
| Excel/CSV 表格 | table        | -              | -               |
| PowerPoint   | presentation | -              | 256             |
| 技术手册         | manual       | DeepDOC        | 512             |
| 法律文档         | laws         | DeepDOC        | 512             |
| 简历           | resume       | DeepDOC        | 256             |
| FAQ 问答       | qa           | -              | -               |
| 图片 OCR       | picture      | -              | -               |
| 简单 PDF       | naive        | Plain Text     | 512             |

### 2.4 解析器源码路径

```python
# 解析器工厂 (rag/svr/task_executor.py:76-93)
FACTORY = {
    "general": naive,
    ParserType.NAIVE.value: naive,
    ParserType.BOOK.value: book,
    ParserType.TABLE.value: table,
}
```

- 解析器工厂：`rag/svr/task_executor.py:76-93`
- General 解析器：`rag/app/naive.py`
- Book 解析器：`rag/app/book.py`
- Table 解析器：`rag/app/table.py`

### 2.5 相关参考文档

- `@reference/文档上传解析文档介绍.md` - SDK API 完整指南
- `@reference/配置embedding模型.md` - Embedding 配置说明

---

## 三、评测系统

### 3.1 两种评测方式对比

| 特性         | benchmark.py                | evaluation_service                      |
| ---------- | --------------------------- | --------------------------------------- |
| **定位**     | 标准数据集测试                     | 自定义测试集评测                                |
| **文件路径**   | `rag/benchmark.py`          | `api/db/services/evaluation_service.py` |
| **数据来源**   | MS MARCO, Trivia QA, MIRACL | 用户上传 JSON/JSONL                         |
| **API 接口** | 命令行工具                       | HTTP API                                |
| **适用场景**   | 模型选型、标准对比                   | 业务效果评估                                  |

### 3.2 检索指标详解

| 指标        | 公式                | 含义     | 标注需求               |
| --------- | ----------------- | ------ | ------------------ |
| Precision | \|R ∩ A\| / \|A\| | 检索精确度  | relevant_chunk_ids |
| Recall    | \|R ∩ A\| / \|R\| | 检索召回率  | relevant_chunk_ids |
| F1 Score  | 2×P×R/(P+R)       | 综合评价   | relevant_chunk_ids |
| Hit Rate  | I(\|R ∩ A\|>0)    | 命中率    | relevant_chunk_ids |
| MRR       | 1/rank₁           | 首位倒数排名 | relevant_chunk_ids |

*R = 相关集合, A = 检索集合*

### 3.3 生成指标详解

**当前支持**：

- 答案长度
- 是否生成答案
- 执行时间
- Token 使用量

**计划中**：

- Faithfulness（忠实度/幻觉检测）
- Answer Relevance（答案相关性）
- Context Relevance（上下文相关性）
- Semantic Similarity（语义相似度）

### 3.4 测试数据格式

**RAGFlow 标准格式**：

```json
{
  "dataset_name": "测试数据集",
  "cases": [
    {
      "question": "测试问题",
      "reference_answer": "参考答案",
      "relevant_doc_ids": ["doc_001"],
      "relevant_chunk_ids": ["chunk_123"],
      "metadata": {
        "doc_type": "table",
        "difficulty": "medium"
      }
    }
  ]
}
```

**批量导入格式** (JSONL)：

```jsonl
{"question": "...", "reference_answer": "...", "relevant_doc_ids": ["..."]}
{"question": "...", "reference_answer": "...", "relevant_doc_ids": ["..."]}
```

### 3.5 评测 API 使用说明

详见 `@reference/评测文档.md`

### 3.6 评测源码路径

```
评估:
├── api/apps/evaluation_app.py          # 评估 API
├── api/db/services/evaluation_service.py  # 评估服务
└── rag/benchmark.py                   # 基准测试工具
```

### 3.7 相关参考文档

- `@reference/评测文档.md` - 评测系统使用指南
- `@reference/评测数据管理.md` - 评测数据管理说明

---

## 四、知识库与元数据管理

### 4.1 知识库模型字段

**核心字段** (`api/db/db_models.py:734-768`):

```python
class Knowledgebase(DataBaseModel):
    id = CharField(primary_key=True)
    name = CharField(max_length=128)
    language = CharField(default="Chinese")
    embd_id = CharField()              # 默认 embedding 模型
    parser_id = CharField()             # 默认解析器
    parser_config = JSONField()         # 默认解析配置
    similarity_threshold = FloatField(default=0.2)
    vector_similarity_weight = FloatField(default=0.3)
```

### 4.2 元数据存储结构

```python
class Document(DataBaseModel):
    meta_fields = JSONField(null=True, default={})
```

**元数据示例**：

```json
{
  "category": "财务",
  "year": "2024",
  "department": "财务部",
  "tags": ["年度", "审计"]
}
```

### 4.3 核心服务函数

| 函数                          | 行号      | 功能      |
| --------------------------- | ------- | ------- |
| `get_meta_by_kbs()`         | 688-714 | 聚合元数据   |
| `get_flatted_meta_by_kbs()` | 718-752 | 展开元数据列表 |
| `get_metadata_summary()`    | 756-777 | 按频率排序汇总 |
| `get_filter_by_kb_id()`     | 173-251 | 提供过滤选项  |

文件位置：`api/db/services/document_service.py`

### 4.4 文档选择机制

**对话级别知识库选择** (`api/apps/dialog_app.py`):

```python
POST /dialog/create
{
  "name": "财务问答",
  "kb_ids": ["kb_财务", "kb_通用"]
}
```

**搜索级别文档过滤** (`rag/nlp/search.py:63-73`):

```python
def get_filters(self, req):
    condition = {}
    if "kb_ids" in req:
        condition["kb_id"] = req["kb_ids"]
    if "doc_ids" in req:
        condition["doc_id"] = req["doc_ids"]
    return condition
```

### 4.5 源码路径

- 元数据服务：`api/db/services/document_service.py:688-777`
- 搜索过滤：`rag/nlp/search.py:63-73`

---

## 五、完整源码路径索引

### 后端核心

```
api/
├── ragflow_server.py              # Flask 应用入口
├── apps/
│   ├── kb_app.py                  # 知识库 API
│   ├── dialog_app.py              # 对话 API
│   ├── document_app.py            # 文档 API
│   ├── evaluation_app.py          # 评估 API
│   └── sdk/
│       ├── dataset.py             # SDK 数据集 API
│       └── doc.py                 # SDK 文档 API
├── db/
│   ├── db_models.py               # 数据库模型
│   └── services/
│       ├── evaluation_service.py  # 评估服务
│       └── document_service.py    # 文档服务（含元数据）
```

### 解析器

```
rag/
├── svr/task_executor.py           # 解析器工厂 (76-93行)
└── app/
    ├── naive.py                   # general 解析器
    ├── book.py                    # book 解析器
    └── table.py                   # table 解析器
```

### 评估

```
rag/
├── benchmark.py                   # 基准测试工具
└── nlp/
    └── search.py                  # 搜索过滤 (63-73行)
```

### 部署

```
docker/
├── docker-compose.yml             # Docker 编排
├── docker-compose-base.yml        # 基础服务
├── .env                           # 环境配置
└── service_conf.yaml.template     # 服务配置模板
```

### 前端

```
web/
├── package.json                   # 前端依赖
└── src/pages/dataset/
    └── dataset-overview/
        └── dataset-filter.tsx     # 数据集过滤组件
```

---

## 六、参考文档索引

### 部署与架构

- `@reference/ragflow中Docker容器说明.md`
- `@reference/RAGFlow服务介绍.md`
- `@reference/源码启动 ragflow 使用说明.md`
- `@reference/安装环境问题.md`

### 文档解析

- `@reference/文档上传解析文档介绍.md`
- `@reference/配置embedding模型.md`

### 评测系统

- `@reference/评测文档.md`
- `@reference/评测数据管理.md`

### 其他

- `@reference/tmux测试开发.md`
- `@reference/RAGflow_9831端口说明.md`

---

*文档版本: v1.1*
*创建日期: 2026-01-16*
*基于 RAGFlow 开源项目*

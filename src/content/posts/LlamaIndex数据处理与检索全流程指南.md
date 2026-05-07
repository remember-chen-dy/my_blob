---
title: LlamaIndex 数据处理与检索全流程指南
published: 2026-05-01
description: LlamaIndex 是一个基于 LLM 的数据处理框架，它提供了从数据加载、预处理、索引构建到查询执行的全流程支持。本文将介绍 LlamaIndex 的使用方法，以及在实际项目中的应用。
tags: [LlamaIndex, LLM]
category: LlamaIndex
draft: false
---


# LlamaIndex 数据处理与检索全流程指南

## 1. 大模型调用

### 1.1 原生 DashScope 调用

阿里云 DashScope 提供了统一的大模型调用接口，支持纯文本和多模态输入。其设计理念是屏蔽不同模型的 API 差异，让开发者通过统一的方式调用通义千问系列模型。

```python
import os
import dashscope

dashscope.base_http_api_url = "https://dashscope.aliyuncs.com/api/v1"

messages = [
    {
        "role": "user",
        "content": [
            {"image": "https://help-static-aliyun-doc.aliyuncs.com/file-manage-files/zh-CN/20241022/emyrja/dog_and_girl.jpeg"},
            {"text": "图中描绘的是什么景象?"}
        ]
    }
]

response = dashscope.MultiModalConversation.call(
    api_key=os.getenv('DASHSCOPE_API_KEY'),
    model='qwen-vl-plus',
    messages=messages
)

if response.status_code == 200:
    print(response.output.choices[0].message.content[0]["text"])
```

核心要点：
- 多模态模型可以同时接收图片 URL 和文本，模型内部会自动对齐视觉和语言特征
- messages 采用标准的对话格式，支持多轮上下文
- 生产环境中务必使用环境变量管理 API Key，避免硬编码泄露

### 1.2 LlamaIndex 封装调用

LlamaIndex 对 DashScope 进行了抽象封装，使得大模型可以无缝接入其数据处理流水线。封装后的 LLM 对象不仅支持基础补全，还支持结构化输出、函数调用和异步调用。

版本兼容性提示：llama_index 新版本广泛使用了 | 联合类型语法（如 str | None），这要求 Python 3.10+。如果你的环境是 Python 3.9，请锁定 llama_index<0.10.0 或升级 Python 版本。

```python
from llama_index.llms.dashscope import DashScope, DashScopeGenerationModels
import os

os.environ["DASHSCOPE_API_KEY"] = "your-api-key"

# 基础文本补全
dashscope_llm = DashScope(model_name=DashScopeGenerationModels.QWEN_MAX)
resp = dashscope_llm.complete("How to make cake?")
print(resp)

# 结构化输出示例
from llama_index.core.tools import FunctionTool, ToolMetadata
from pydantic import BaseModel

class SendModel(BaseModel):
    """发送短信模型"""
    phone: str
    content: str

def send_mes(phone: str, content: str) -> dict:
    """发送短信工具函数"""
    return {"status": "success", "phone": phone, "content": content}

tool = FunctionTool.from_defaults(
    fn=send_mes,
    tool_metadata=ToolMetadata(
        name="发送短信工具",
        description="发送短信",
        fn_schema=SendModel
    )
)

# 结构化预测：让 LLM 按照 Pydantic Schema 提取信息
from llama_index.core import PromptTemplate

prompt_template = PromptTemplate("请从下面的文本提取信息: {text}")
response = dashscope_llm.structured_predict(
    SendModel,
    prompt_template,
    text="我的手机号是13800138000，信息是明天见"
)
print(f"结构化输出: {response}")
```

封装层的设计优势：
- 统一接口：无论底层是 DashScope、OpenAI 还是本地模型，上层调用方式一致
- 结构化输出：structured_predict 利用 Pydantic 模型约束 LLM 输出格式，自动完成 JSON Schema 校验
- 工具集成：FunctionTool 将 Python 函数转化为 LLM 可调用的工具，使模型具备行动能力
- 异步支持：所有模型都提供 acomplete、achat 等异步方法，适合高并发场景

## 2. 数据分割策略

数据分割（Chunking）是 RAG 系统的基础环节。分割质量直接决定了检索粒度和答案完整性。过大的块包含太多无关信息会稀释语义浓度，过小的块则会丢失上下文连贯性。

### 2.1 基于 Token 的分割

TokenTextSplitter 是最基础的分割器，按照 token 数量进行机械切割。它不关心语义边界，纯粹从文本长度维度进行切分。适用场景：文本语义均匀、无明显段落结构的场景。

```python
from llama_index.core.node_parser import TokenTextSplitter
from llama_index.core import Document

documents = [
    Document(
        text="小说是一种叙事性的文学体裁，通过人物的塑造和情节、环境的描述来概括地表现社会生活。"
             "人物、情节、环境是小说的三要素。情节一般包括开端、发展、高潮、结局四部分。"
             "小说与诗歌、散文、戏剧，并称四大文学体裁。",
        metadata={'source': "文学百科"}
    )
]

token_splitter = TokenTextSplitter(
    chunk_size=50,        # 每块最多 50 个 token
    chunk_overlap=10,     # 块之间重叠 10 个 token
    separator=" ",        # 首选分隔符
    backup_separators=["。", "，", ".", ",", "!", "?"]  # 备用分隔符
)

nodes = token_splitter.get_nodes_from_documents(documents)
for i, node in enumerate(nodes):
    print(f"Chunk {i}: {node.text[:50]}...")
```

参数详解：
- chunk_size：每个文本块的目标 token 数，实际大小会因分隔符位置略有浮动
- chunk_overlap：相邻块之间的重叠 token 数，用于保持上下文连贯。设置为 0 可能导致关键信息被截断
- separator：首选分隔符，分割器会优先在此处断开
- backup_separators：当首选分隔符无法满足大小时，依次尝试备用分隔符。中文场景建议加入中文标点

### 2.2 基于语义的分割

语义分割器利用 Embedding 模型计算相邻句子之间的语义相似度，当相似度骤降时标记为语义边界。这种方式更贴近人类理解文本的自然方式，适用场景：主题切换频繁、段落分明的结构化文档。

```python
from llama_index.embeddings.dashscope import DashScopeEmbedding
from llama_index.core.node_parser import SemanticSplitterNodeParser

embed_model = DashScopeEmbedding(
    model_name='text-embedding-v4',
    api_key="your-api-key"
)

semantic_splitter = SemanticSplitterNodeParser(
    buffer_size=1,                          # 滑动窗口大小
    breakpoint_percentile_threshold=95,      # 分割点阈值（百分位）
    embed_model=embed_model
)

nodes = semantic_splitter.get_nodes_from_documents(documents)
for i, node in enumerate(nodes):
    print(f"Semantic Chunk {i}: {node.text[:80]}...")
```

工作原理详解：
语义分割器的核心是滑动窗口 + 相似度下降检测：
1. 将文本按句子拆分
2. 使用 buffer_size 控制窗口大小，计算窗口内相邻句子的余弦相似度
3. 形成一个相似度序列，计算序列的百分位数
4. 相似度低于 breakpoint_percentile_threshold 百分位的地方被标记为分割点

```
句子1 ── 相似度0.92 ── 句子2 ── 相似度0.88 ── 句子3 ── 相似度0.35(分割点!) ── 句子4 ── 相似度0.90 ── 句子5
```

参数调优建议：
- buffer_size 越大，窗口包含的上下文越丰富，但对局部变化不敏感
- breakpoint_percentile_threshold 越高（如 99），分割点越少，每块越大；越低（如 90），分割越细碎
- 建议先用少量文档测试，观察分隔效果后调整

### 2.3 层次文档分割

层次分割先识别文档的结构层级（如标题 # ## ###、章节编号），按高层级进行粗分割，再对每个子部分进行细粒度分割。这种方式保留了文档的层级关系，每个 chunk 都知道自己的父节点和子节点。

```python
from llama_index.core.node_parser import HierarchicalNodeParser

hierarchical_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],   # 三个层级的大小：大块→中块→小块
    chunk_overlap=20
)

nodes = hierarchical_parser.get_nodes_from_documents(documents)

# 每个 node 会包含 parent_node 和 child_nodes 关系
for node in nodes:
    print(f"层级关系 - 父节点: {node.parent_node}, 子节点数: {len(node.child_nodes) if node.child_nodes else 0}")
```

分层结构示意：
```
顶层 (2048 tokens)
├── 中层A (512 tokens)
│   ├── 底层1 (128 tokens)
│   └── 底层2 (128 tokens)
└── 中层B (512 tokens)
    ├── 底层3 (128 tokens)
    └── 底层4 (128 tokens)
```

检索优势：
- 可以先匹配底层小块获得高精度结果
- 如果小块上下文不足，自动向上追溯到父节点获取更大上下文
- 特别适合长文档（如书籍、技术手册）的结构化问答

## 3. 元数据处理

元数据（Metadata）为每个文本块附加结构化信息，在检索时可用于过滤、排序和上下文增强。LlamaIndex 提供了自动元数据提取器，可以从文本中自动抽取标题、关键词和摘要。

```python
from llama_index.core import Document
from llama_index.core.extractors import (
    TitleExtractor,
    KeywordExtractor,
    SummaryExtractor
)

documents = [
    Document(
        text="机器学习是人工智能的一个分支，它使计算机能够从数据中学习并做出预测或决策，"
             "而无需进行明确的编程。常见的机器学习类型包括监督学习、无监督学习和强化学习。",
        metadata={
            "source": "AI教程",
            "author": "张三",
            "date": "2025-01-15",
            "category": "技术"
        }
    )
]

# 自动提取器组合
extractors = [
    TitleExtractor(nodes=5),          # 提取/生成文档标题
    KeywordExtractor(keywords=5),     # 提取5个关键词
    SummaryExtractor(summaries=["self"])  # 生成自摘要
]

from llama_index.core.ingestion import IngestionPipeline

pipeline = IngestionPipeline(
    transformations=[
        TokenTextSplitter(chunk_size=256),
        *extractors  # 元数据提取器作为转换步骤加入管道
    ]
)

nodes = pipeline.run(documents=documents)

for node in nodes:
    print(f"提取的标题: {node.metadata.get('document_title')}")
    print(f"提取的关键词: {node.metadata.get('excerpt_keywords')}")
```

元数据的实际价值：
- 过滤查询：只搜索特定日期范围、作者或类别的文档
- 相关性排序：关键词匹配度可以作为排序因子
- 答案溯源：生成的答案可以附带来源元数据，增强可解释性

## 4. 数据转换与预处理

IngestionPipeline 是 LlamaIndex 的数据处理核心，它将分割、元数据提取、向量化等步骤串联为一条可配置、可复用的流水线。设计理念类似于 ETL 管道。

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.dashscope import DashScopeEmbedding

# 构建完整的预处理管道
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=50),  # 分句分割
        DashScopeEmbedding(model_name='text-embedding-v4'),   # 向量化
    ]
)

documents = [Document(text="你的文档内容...")]
nodes = pipeline.run(documents=documents)
print(f"处理完成，共生成 {len(nodes)} 个节点")
```

管道设计的优势：
- 模块化：每个步骤可独立替换（如更换分割器或 Embedding 模型）
- 缓存机制：管道支持节点缓存，重复文档不会重新处理
- 批量处理：支持异步批量处理大量文档

## 5. 数据存储与查询

向量存储是 RAG 系统的持久化层。LlamaIndex 支持多种向量数据库，从轻量级的内存存储到生产级的分布式数据库。

```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.vector_stores.chroma import ChromaVectorStore
import chromadb

# 创建 Chroma 向量存储（持久化到磁盘）
chroma_client = chromadb.PersistentClient(path="./chroma_db")
chroma_collection = chroma_client.create_collection("my_documents")
vector_store = ChromaVectorStore(chroma_collection=chroma_collection)

# 构建存储上下文和索引
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex(nodes, storage_context=storage_context)

# 持久化索引元数据
index.storage_context.persist(persist_dir="./storage")

# 查询 - similarity_top_k 控制返回的最相似文档块数量
query_engine = index.as_query_engine(similarity_top_k=3)
response = query_engine.query("什么是机器学习？")
print(response)
```

向量数据库选型建议：

| 数据库 | 适用场景 | 特点 |
|--------|----------|------|
| Chroma | 开发/小规模部署 | 轻量、易用、支持持久化 |
| Qdrant | 中等规模 | 高性能、丰富过滤 |
| Milvus | 大规模生产 | 分布式、高可用 |
| Pinecone | 无服务器场景 | 免运维、按量付费 |

## 6. 索引与高级检索

### 6.1 核心数据流

LlamaIndex 的数据处理遵循一条清晰的流水线：

```
Document ──→ Node ──→ Index ──→ Retriever ──→ Query Engine ──→ Response
```

- Document：原始数据载体
- Node：经过分割的文本块，是检索的基本单位
- Index：数据结构的组织方式，决定检索策略
- Retriever：根据查询从索引中提取相关节点
- Query Engine：将检索结果与 LLM 结合生成最终答案

### 6.2 查询响应模式

不同的响应模式影响 LLM 如何消化检索到的多个文本块：

| 模式 | 原理 | 适用场景 |
|------|------|----------|
| refine | 顺序精炼：将第一个块和问题生成初始答案，然后用后续块逐个精炼 | 高精度需求，能充分利用每个块 |
| compact | 压缩拼接：将多个块拼接为一个 prompt（超长时压缩），一次性生成答案 | 效率优先，降低 API 调用次数 |
| tree_summarize | 树形汇总：对块进行分组→总结→再分组，自底向上生成答案 | 大量文档的综合分析 |
| accumulate | 逐步累积：逐步追加上下文，适合长文本理解 | 需保留完整上下文的场景 |
| no_text | 不传递文本：仅用 LLM 自身知识回答 | 无需检索的纯推理 |

```python
# 不同响应模式的实际使用

# refine 模式：最耗 token，但答案最细致
query_engine_refine = index.as_query_engine(response_mode="refine")

# compact 模式：最省 token，适合简单问答
query_engine_compact = index.as_query_engine(response_mode="compact")

# tree_summarize 模式：适合"总结所有文档"类问题
query_engine_tree = index.as_query_engine(response_mode="tree_summarize")
```

### 6.3 SummaryIndex（摘要索引）

原理：将所有文本块以线性列表存储。查询时遍历所有节点，使用 LLM 生成一个综合答案。本质是穷举式检索。

适用场景：文档数量少（<100个节点），需要对全部内容做总结或提取。

```python
from llama_index.core import SummaryIndex

summary_index = SummaryIndex(nodes)
query_engine = summary_index.as_query_engine(response_mode="tree_summarize")
response = query_engine.query("总结所有文档的主要内容")
print(response)
```

优缺点：
- 不会遗漏任何信息
- 节点数量多时耗 token 巨大，响应缓慢

### 6.4 DocumentSummaryIndex（文档摘要索引）

两阶段检索策略：
- 摘要匹配：为每份文档生成一个高质量摘要并向量化，查询时先匹配摘要
- 内容深入：对匹配到的文档，再深入其完整内容生成答案

这种"先筛选后精读"的模式模拟了人类快速翻阅文档目录的行为。

```python
from llama_index.core import DocumentSummaryIndex

doc_summary_index = DocumentSummaryIndex.from_documents(
    documents,
    llm=dashscope_llm,
    embed_model=embed_model,
    show_progress=True
)

query_engine = doc_summary_index.as_query_engine(
    response_mode="tree_summarize",
    verbose=True
)
response = query_engine.query("小说的三要素是什么？")
print(response)
```

最佳实践：
- 文档摘要在索引构建时生成（耗时较高），查询时无需重复
- 适合文档数量适中（数百份）但每份文档较长的场景
- 摘要质量直接影响检索精度，建议使用强模型生成摘要

### 6.5 KeywordTableIndex（关键词表索引）

原理：在索引构建阶段从每个节点提取关键词，建立"关键词→节点"的映射表。查询时提取问题关键词，直接匹配对应节点。

```python
from llama_index.core import KeywordTableIndex

keyword_index = KeywordTableIndex.from_documents(
    documents,
    llm=dashscope_llm
)

query_engine = keyword_index.as_query_engine()
response = query_engine.query("什么是小说的三要素？")
print(response)
```

适用场景：
- 文档内容有明显关键词特征的场景
- 配合其他索引做混合检索

限制：
- 严重依赖关键词提取质量
- 无法处理同义词（除非做扩展，如结合 LLMSynonymRetriever）

### 6.6 TreeIndex（树形索引）

结构：自底向上构建树形结构，叶子节点是原始文本块，上层节点是 LLM 总结的摘要。检索时从根节点出发，由 LLM 引导选择深入路径。

```
          根节点（所有内容的摘要）
     ┌──────────┬──────────┐
  中层摘要A  中层摘要B  中层摘要C
   ┌──┴──┐    ┌──┴──┐    ┌──┴──┐
  块1  块2   块3  块4   块5  块6
```

Select Leaf 查询模式：
1. 从根节点开始
2. 向 LLM 发送当前节点的子节点摘要
3. LLM 判断哪个子节点与问题最相关
4. 递归深入直到叶子节点

```python
from llama_index.core import TreeIndex

tree_index = TreeIndex.from_documents(
    documents,
    llm=dashscope_llm,
    num_children=3  # 每个节点最多3个子节点
)

query_engine = tree_index.as_query_engine(
    response_mode="tree_summarize",
    verbose=True
)
response = query_engine.query("小说的起源是什么？")
print(response)
```

优势：
- 检索效率高，不需要遍历所有节点
- 每个摘要层提供了不同粒度的上下文

代价：
- 构建成本高（需要 LLM 为每个中间节点生成摘要）
- 如果某层摘要误导了 LLM，可能导致错误路径

### 6.7 PropertyGraphIndex（属性图索引）

核心概念：将文档中的实体和关系提取为图结构，每个节点和边都可以携带属性。这是最灵活但也最复杂的索引类型。

```
[华晨宇] --(演唱)--> [好想爱这个世界啊]
    |                      |
  (出生地)             (发行年份)
    |                      |
  [湖北]                [2019年]
```

构建流程：
1. 实体识别：从文档中提取实体作为图节点
2. 关系抽取：识别实体间关系作为图的边
3. 属性提取：为节点和边附加属性信息
4. 向量化：对节点文本做 Embedding 存入向量库

```python
from llama_index.core import PropertyGraphIndex
from llama_index.core.graph_stores import SimplePropertyGraphStore

# 创建图存储
graph_store = SimplePropertyGraphStore()

# 构建知识图谱索引
pg_index = PropertyGraphIndex.from_documents(
    documents,
    llm=dashscope_llm,
    embed_model=embed_model,
    property_graph_store=graph_store,
    show_progress=True
)

# 向量上下文检索：基于语义相似度找到相关节点
from llama_index.core.indices.property_graph import VectorContextRetriever

vector_retriever = VectorContextRetriever(
    pg_index.property_graph_store,
    vector_store=pg_index.vector_store,
    similarity_top_k=2
)

results = vector_retriever.retrieve("小说体裁的分类")
for result in results:
    print(f"检索结果: {result.text[:100]}...")

# LLM 同义词检索：利用大模型扩展查询词的同义词
from llama_index.core.indices.property_graph import LLMSynonymRetriever

synonym_retriever = LLMSynonymRetriever(
    pg_index.property_graph_store,
    llm=dashscope_llm,
    num_synonyms=3
)

results = synonym_retriever.retrieve("文学体裁")
for result in results:
    print(f"同义词扩展结果: {result.text[:100]}...")
```

混合检索策略：
PropertyGraphIndex 支持将多种检索器组合使用：
- VectorContextRetriever：处理模糊语义查询
- LLMSynonymRetriever：扩展关键词匹配
- 自定义检索器：基于图算法的路径检索

适用场景：
- 知识图谱类应用（人物关系、产品关联）
- 需要多跳推理的复杂问答
- 结构化数据的检索增强

## 7. 索引选型对比

| 索引类型 | 构建成本 | 检索精度 | 适用文档量 | 核心优势 |
|----------|----------|----------|------------|----------|
| VectorStoreIndex | 低 | 中-高 | 任意 | 通用性强，开箱即用 |
| SummaryIndex | 低 | 高 | <100 | 不会遗漏信息 |
| DocumentSummaryIndex | 高 | 高 | 数百 | 两阶段精读，摘要先行 |
| KeywordTableIndex | 中 | 中 | 中 | 关键词精确匹配 |
| TreeIndex | 高 | 高 | 大 | 层次化检索，效率高 |
| PropertyGraphIndex | 最高 | 最高 | 中等 | 实体关系建模，多跳推理 |

## 8. 总结与最佳实践

- 从简开始：先用 VectorStoreIndex 快速验证，根据效果迭代到复杂索引
- 分割是基础：选对分割策略比选对索引类型更重要，中文场景优先考虑语义分割
- 元数据不可少：结构化元数据是检索过滤的关键，花时间设计好的元数据结构
- 响应模式因地制宜：简单问答用 compact，综合分析用 tree_summarize
- 混合检索是趋势：单一检索策略往往不够，PropertyGraphIndex 的混合检索代表了前进方向
- 持续优化：RAG 系统需要根据实际查询日志持续调优分割参数和检索策略

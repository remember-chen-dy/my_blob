---
title: LangChain：用链式调用大模型、数据与工具串联
published: 2026-04-01
updated: 2026-04-29
description: '本文从零开始介绍 LangChain 的基础用法，涵盖模型调用、提示模板、链式组合、记忆管理、工具与代理等核心概念，配合可运行的代码示例与对比表格，帮助开发者快速构建第一个 LLM 驱动的应用。'
image: ''
tags: [LangChain, Example]
category: 'LangChain '
draft: false 
---

# 5分钟上手LangChain：用链式调用把大模型、数据与工具串联起来

> 本文从零开始介绍 LangChain 的基础用法，涵盖模型调用、提示模板、链式组合、记忆管理、工具与代理等核心概念，配合可运行的代码示例与对比表格，帮助开发者快速构建第一个 LLM 驱动的应用。

## 一、什么是 LangChain

LangChain 是一个专为大语言模型（LLM）应用开发设计的开源框架。它的核心理念是“链式调用”（chaining）：将大模型、外部数据源、工具、记忆等组件像搭积木一样串联成一条可执行的处理流水线。无论是简单的问答机器人，还是需要多步推理的智能代理，都可以通过 LangChain 的模块化设计快速实现。

随着 GPT-4、Claude、Gemini 等大模型的爆发，开发者面临的不再是“如何调用模型”，而是“如何让模型可靠地完成复杂任务”。LangChain 解决了以下几个痛点：

- **连接外部世界**：让 LLM 访问实时数据、私有知识库、API。
- **记忆管理**：为无状态的 API 调用注入对话历史与上下文。
- **工作流编排**：将多个处理步骤（检索、推理、证据验证、工具调用）组合成自动化链条。
- **模型无关性**：屏蔽不同模型供应商的接口差异，轻松切换底层模型。

本篇文章将带你从安装开始，逐一体验这些核心模块，并通过可运行的 Python 代码完成第一个 LangChain 应用。

## 二、环境准备

确保 Python ≥ 3.9，然后安装 LangChain 及 OpenAI 适配器：

```bash
pip install langchain langchain-openai
```

设置 OpenAI API Key 环境变量（也可在代码中直接传入）：

```bash
export OPENAI_API_KEY="your-api-key-here"
```

基础代码中，我们用 `ChatOpenAI` 作为大模型入口：

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)
response = llm.invoke("用一句话解释 LangChain")
print(response.content)
```

## 三、模型 I/O：提示模板与输出解析

大模型应用的本质是 **输入→模型→输出** 的流水线，LangChain 将其抽象为三个核心环节：

| 组件               | 作用                               | 常用类                       |
| ------------------ | ---------------------------------- | ---------------------------- |
| 提示模板 (Prompt)  | 将动态变量嵌入固定提示结构         | `PromptTemplate`, `ChatPromptTemplate` |
| 模型 (LLM / Chat)  | 接收提示，生成文本或消息           | `ChatOpenAI`, `HuggingFaceEndpoint` |
| 输出解析器 (Parser)| 将模型自由文本转化为结构化数据     | `StrOutputParser`, `PydanticOutputParser` |

示例：用提示模板生成餐厅介绍，并返回纯字符串。

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template(
    "写一句关于{restaurant}的吸引人广告语，突出它的{feature}。"
)

parser = StrOutputParser()
chain = prompt | llm | parser

result = chain.invoke({"restaurant": "海底捞", "feature": "极致服务"})
print(result)   # 输出：来海底捞，感受让每个人都笑出声的极致服务！
```

这里使用 LCEL（LangChain Expression Language）的管道操作符 `|` 串联各个组件，使得数据流一目了然。

## 四、链（Chains）：让组件流动起来

在 LangChain 中，`Chain` 是多个可运行单元的组合。除了上面的简单管道，我们还可以定义顺序执行的多步链、分支链、并行链等。最基础的链是 `RunnableSequence`，由管道符自动创建。

但显式地使用旧版 `LLMChain` 依然值得了解，因为它内部自动绑定了提示、模型和输出解析器：

```python
from langchain.chains import LLMChain
from langchain_core.prompts import PromptTemplate
from langchain_openai import OpenAI   # 旧版 LLM 接口

llm_old = OpenAI(temperature=0.7)
prompt = PromptTemplate(
    input_variables=["topic"],
    template="请为关于{topic}的博客文章起三个标题。"
)

chain_old = LLMChain(llm=llm_old, prompt=prompt)
print(chain_old.run("Python 异步编程"))
```

不过，LangChain 目前推荐使用 LCEL 管道式语法。下面的示例展示如何将多个步骤串联成一条链：

```python
from langchain_core.runnables import RunnableLambda

def upper_case(input: str) -> str:
    return input.upper()

chain = (
    prompt
    | llm
    | parser
    | RunnableLambda(upper_case)
)

print(chain.invoke({"topic": "机器学习"}))
```

## 五、记忆（Memory）：记住对话上下文

默认情况下，LLM 的每次 API 调用都是无状态的。要实现多轮对话，需要自己管理历史消息。LangChain 提供了多种 `Memory` 类，可以自动将历史记录插入到提示中。

最简单的对话记忆是 `ConversationBufferMemory`：

```python
from langchain.memory import ConversationBufferMemory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables import RunnablePassthrough

memory = ConversationBufferMemory(return_messages=True)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个友好的助手"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

chain = (
    {
        "history": lambda x: memory.load_memory_variables(x)["history"],
        "input": lambda x: x["input"]
    }
    | prompt
    | llm
    | parser
)

def chat(message):
    response = chain.invoke({"input": message})
    memory.save_context({"input": message}, {"output": response})
    return response

print(chat("我叫张三"))
print(chat("我叫什么名字？"))   # 模型会记住你叫张三
```

`MessagesPlaceholder` 会在提示中的指定位置插入历史消息列表。`RunnablePassthrough` 则负责将输入原样传递，这里我们自定义了字典生成函数来收集 `history` 和 `input`。

当然，LangChain 还支持摘要记忆（只保留对话摘要）、缓冲窗口记忆（只保留最近 k 轮）等，适合不同场景的长对话管理。

## 六、检索增强生成（RAG）：给模型装上知识库

LLM 的训练数据有截止日期，且无法访问企业内部文档。RAG（Retrieval-Augmented Generation）的思路是：先从外部知识库中检索相关文档，再把文档和问题一起发给模型回答问题。LangChain 为此提供了一整套数据连接模块。

### 6.1 文档加载与分割

```python
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

loader = TextLoader("knowledge.txt")   # 假设有一份本地文本
documents = loader.load()

splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=50)
chunks = splitter.split_documents(documents)
print(f"文档切分为 {len(chunks)} 块")
```

### 6.2 向量化与存储

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_documents(chunks, embeddings)
```

### 6.3 构建 RAG 链

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.prompts import ChatPromptTemplate

retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

template = """根据以下上下文回答问题：
{context}

问题：{question}"""
prompt = ChatPromptTemplate.from_template(template)

chain = (
    {
        "context": lambda x: "\n\n".join(
            [doc.page_content for doc in retriever.invoke(x["question"])]
        ),
        "question": lambda x: x["question"]
    }
    | prompt
    | llm
    | parser
)

answer = chain.invoke({"question": "公司今年的战略目标是什么？"})
print(answer)
```

至此，一个最简单的“文档问答机器人”就完成了。你可以把 `knowledge.txt` 换成任何产品手册、规章制度，模型都能基于最新内容作答。

## 七、工具与代理（Tools & Agents）：让模型学会“用工具”

LLM 本身只能生成文本，不能发送邮件、查询数据库或调用 API。但我们可以给模型提供一个工具列表，并让它自主判断何时调用哪个工具。这就是 **Agent（代理）** 的思想：模型作为推理引擎，工具作为执行手脚。

LangChain 内置了大量现成工具：搜索引擎、计算器、维基百科、SQL 查询等。下面展示如何创建一个能自主使用计算器和搜索工具的代理。

```python
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_community.tools import TavilySearchResults
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# 自定义一个简单的计算工具
@tool
def multiply(a: float, b: float) -> float:
    """计算两个数的乘积"""
    return a * b

# 搜索工具（需要 Tavily API Key，也可用 DuckDuckGo 等）
search = TavilySearchResults(max_results=2)

tools = [multiply, search]
llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

prompt = """你是一个能使用工具的助手。如果工具无法解答，请直接回答。
可用工具：{tools}
使用以下格式：
问题：必须回答的输入
思考：你该做什么
行动：工具名称
行动输入：工具参数
观察：工具返回结果...（继续思考/行动/观察，直到得到最终答案）
最终答案：对问题的最终回复

开始！
问题：{input}
思考：{agent_scratchpad}"""

# 使用新版工具调用代理
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({"input": "2024年奥运会举办城市人口乘以2是多少？"})
print(result["output"])
```

当用户提问时，代理会先调用搜索工具获取城市人口，再将结果传给乘法工具计算，最终返回答案。整个过程完全自动，开发者只需定义工具和代理即可。

## 八、LangChain 核心模块总结

为了方便记忆和查阅，下表汇总了 LangChain 最常用的模块及其功能：

| 模块               | 主要功能                                     | 代表性类/函数                                      |
| ------------------ | -------------------------------------------- | -------------------------------------------------- |
| 模型 I/O           | 管理提示、调用模型、格式化输出               | `ChatPromptTemplate`, `ChatOpenAI`, `StrOutputParser` |
| 链 (Chains)        | 串联多个可运行单元，形成端到端流水线         | `RunnableSequence`, `LLMChain`, 管道 `\|`         |
| 记忆 (Memory)      | 存储对话历史、上下文                         | `ConversationBufferMemory`, `ConversationSummaryMemory` |
| 数据连接           | 加载文档、文本分割、向量化、检索             | `TextLoader`, `RecursiveCharacterTextSplitter`, `FAISS`, `Retriever` |
| 代理与工具         | 让 LLM 动态调用外部工具完成复杂任务          | `create_tool_calling_agent`, `AgentExecutor`, `tool` 装饰器 |
| 回调 (Callbacks)   | 监控、日志、流式输出                         | `BaseCallbackHandler`, `StreamingStdOutCallbackHandler` |

## 九、最佳实践与注意事项

1. **模型选择**：原型验证用 `gpt-3.5-turbo`，追求质量用 `gpt-4`；私有部署可用 `HuggingFaceEndpoint` 或本地模型。
2. **提示工程**：使用 `ChatPromptTemplate` 明确区分系统、用户和 AI 消息，角色设定放在系统消息中。
3. **分块策略**：RAG 中文档分块大小建议 300-500 字符，重叠 50-100 字符，保证语义连贯。
4. **记忆管理**：短对话用 `ConversationBufferMemory`，长对话使用 `ConversationSummaryMemory` 控制 Token 消耗。
5. **安全与成本**：开启 API 调用的速率限制，避免代理无限循环（设置 `max_iterations`）。
6. **调试**：将 `verbose=True` 传入 AgentExecutor 或链，可以打印每一步的中间结果，方便排查问题。

## 十、结语

LangChain 让大模型应用从“单次问答”进化为“可记忆、会检索、能使用工具”的智能体。通过本文的讲解，你应该已经掌握了：

- 如何使用提示模板和输出解析器构建模型输入输出；
- 如何用链式语法串联多个处理步骤；
- 如何加入记忆实现多轮对话；
- 如何接入向量数据库打造 RAG 文档问答；
- 如何通过代理让模型使用外部工具。

所有这些能力都以模块化的方式提供，你可以像搭积木一样组合它们，快速响应业务需求。下一步，建议你动手将文中的代码在本地跑通，并尝试替换模型、工具、知识库，探索 LangChain 更广阔的生态（如 LangServe 部署 API、LangSmith 监控调试等）。

大模型只是引擎，而 LangChain 是给引擎装上方向盘、轮子和导航系统的最佳工具箱。现在，就打开你的 IDE，写出第一个能“串联世界”的智能应用吧！

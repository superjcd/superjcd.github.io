---
title: RAG进阶技巧
date: 2024-04-29
categories:
    - LLM
tags:
    - RAG
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

先简单讲一下什么是RAG(Retrieval-Augmented Generation), 或者说为什么需要RAG。  
LLM的训练数据是基于过去的信息， LLM没有办法回答最新的问题， 比如你没有办法让一个2022年训练的模型回答2023年NBA的冠军归属。  
当然我们可以选择给他喂最新的数据重新训练大模型， 但是这种方式的成本非常高昂。  
如何教"过时"的模型以新知识， 通过RAG， 我们把新的知识灌输或者填充到对话的语境中， 那么LLM通过检索这些新知识， 结合模型原有的理解能力， 他就能够回答自己在训练时没有遇到过的问题。  
本教程我们会使用llamaindex，具体版本信息：  
```
Name: llama-index
Version: 0.10.31
Summary: Interface between LLMs and your data
Home-page: https://llamaindex.ai
Author: Jerry Liu
Author-email: jerry@llamaindex.ai
License: MIT
```

## 基础RAG流程
这里我们会使用llama3而不是OpenAI的chatgpt来作为我们的基座模型， 关于如何部署和使用llama3可以参考ollama的[github文档](https://github.com/ollama/ollama)：

首先， 我们会为我们的RAG应用准备好llm基座和词嵌入模型：
```python
from llama_index.core import Settings
from llama_index.llms.ollama import Ollama
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.core import SimpleDirectoryReader


llm = Ollama(model="llama3", request_timeout=60.0, temperature=0.1)
Settings.llm = llm
embed_modle =  HuggingFaceEmbedding("BAAI/bge-base-en-v1.5")
Settings.embed_model = embed_modle
```
> Settings是全局的配置类， 这样可以免去后续显式输入llm参数以及embed_model参数的过程；这里我们用智源（BAAI）的bge通用模型, 他们的词嵌入模型是比较好的

接着我们要加载文档， 并对文档进行分割
```python
from llama_index.core import SimpleDirectoryReader
from llama_index.core.node_parser import SimpleNodeParser

documents = SimpleDirectoryReader(
      input_files=["./data/paul_graham_eassay.txt"]
).load_data()

node_parser = SimpleNodeParser(chunk_size=1024)
nodes = node_parser.get_nodes_from_documents(documents)

print(len(nodes))
```
我们读取了data路径下的paul_graham_eassay.txt文档， 并将文档分割成了长度为1024的若干个文档， 这样文档数从原来的1个变成了21个
打印其中一个文档：
```python
print(str(nodes[0]))
```
输出：
```
('Node ID: 00886ada-1a64-47ea-9f91-0eb48caf2138\n'
 'Text: What I Worked On    February 2021    Before college the two main\n'
 'things I worked on, outside of school, were writing and programming. I\n'
 "didn't write essays. I wrote what beginning writers were supposed to\n"
 'write then, and probably still are: short stories. My stories were\n'
 'awful. They had hardly any plot, just characters with strong feelings,\n'
 'whic...')
```

在我正式使用语义进行文档查询之前， 我们需要准备好我们的向量数据库， 可选项有很多， 具体可以参考[这里](https://docs.llamaindex.ai/en/stable/community/integrations/vector_stores/);这里我们使用weaviate(推荐使用docker本地运行weaviate， 具体参考[这里](https://weaviate.io/developers/weaviate/installation/docker-compose), )
```python
import weaviate  # v3 version
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.vector_stores.weaviate import WeaviateVectorStore

index_name = "MyExternalContext"
client = weaviate.Client(url="http://localhost:8080")
vector_store  = WeaviateVectorStore(weaviate_client=client, index_name=index_name)

storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex(nodes, storage_context=storage_context)
```
接着我使用VectotreStoreIndex作为我们查询引擎来查询文档：
```python
Question = "What happened at Interleaf?"

query_engine = index.as_query_engine()

response = query_engine.query(Question)

print(response)
```


## 前置过程优化 -- 使用SentenceWindowNodeParser分割文档
所谓的前置优化， 就是发生在query之前的优化。  
前面我们使用了`SimpleNodeParser`来分割我们的文档， 当时我们把文档分割成了长度为1024的21个文档， 这里的问题在于：当我们进行query的时候， 我们会先定位到这21个文档中的某一个来填充llm的context， 然后通过llm来输出问题；  
很显然， 1024这个长度其实有点太大了，不利于精准定位；但是假设我缩小文档的长度，文档匹配的精度会上升， 但是同时可能导致context的上下文信息变少，也利于输出正确的答案；  
所以我们需要在匹配精度和保证context上下文的基础上进行平衡， 这里我们使用 `SentenceWindowNodeParser`来分割文档：
```python
from llama_index.core.node_parser import SentenceWindowNodeParser

node_parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,
    window_metadata_key="window",   # 会成为元数据
    original_text_metadata_key="original_text",
)

nodes = node_parser.get_nodes_from_documents(documents)

print(len(nodes))

vector_store  = WeaviateVectorStore(weaviate_client=client, index_name=index_name)

storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex(nodes, storage_context=storage_context)  # 考虑文
```
同时我们需要去置换`as_query_engine`的`node_postprocessors`参数
```python
from llama_index.core.postprocessor import MetadataReplacementPostProcessor

# The target key defaults to `window` to match the node_parser's default
postproc = MetadataReplacementPostProcessor(
    target_metadata_key="window"
)

query_engine = index.as_query_engine( 
    node_postprocessors = [postproc],
)


response =  query_engine.query(Question)

print(response)
```
## 匹配机制优化--使用Hybrid Search
前面的优化方式， 发生在匹配之前， 我们也可以选择优化匹配算法， 将基于语义（词向量）的方式修改为结合关键词查询
> 关键词查询的特点会查找和查询内容更相似的内容， 比如用户搜索apple， 在关键词查询的场景下， 所有带有apple的文档都是目标文档， 不论apple指的是苹果还是手机。 另外假设用户输入的词语发生了拼写错误， 关键词匹配机制也会失效

```python
query_engine = index.as_query_engine(
    node_postprocessors = [postproc],
    vector_store_query_mode="hybrid", 
    alpha=0.5
) 

response = query_engine.query(Question)

print(response)
```
alpha是0-1之间的值，表示关键词匹配的比例， 如果是0的话， 则不使用；
> 注意并不是所有向量数据库都支持关键词匹配

## 后置过程优化
最后， 当我们得到了目标结果， 如果我们想要进步一步优化查询， 那该怎么处理？
一种方法就是使用reranker从候选文档（比如有6个候选文档）中筛选出1-2个更匹配的文档：

```python
from llama_index.core.postprocessor import SentenceTransformerRerank  # 计算相似度

rerank = SentenceTransformerRerank(top_n=2, model="BAAI/bge-reranker-base")

query_engine = index.as_query_engine(
    similarity_top_k=6,
    node_postprocessors = [rerank, postproc],
    vector_store_query_mode="hybrid", 
    alpha=0.3
)

response = query_engine.query(Question)

print(response)
```

## 参考
[Retrieval-Augmented Generation (RAG): From Theory to LangChain Implementation](https://towardsdatascience.com/retrieval-augmented-generation-rag-from-theory-to-langchain-implementation-4e9bd5f6a4f2)

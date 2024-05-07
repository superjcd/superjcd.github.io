---
title: 高级Prompt优化技巧-基于元提示
date: 2024-04-30
categories:
    - LLM
tags:
    - 提示工程
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
我们知道好的prompt会直接关系到LLM应用的质量，那么如何通过提示工程获得更好的Prompt会是一件非常重要且有意义的事情。  
在实现某种调优之前， 我们首先需要搞清楚一件事情， 如何去评估某一项对Prompt进行调优的工程是不是真的是有效的
## 实现对Query Engine的评估
实现的方式说起来很简单，以QA场景为例，  假定我们有一个人为标注过的数据集，它包含Question列和Answer列（见下表）， 然后我们用LLM回答Question，得到一个预测回答， 然后对照真实Answer和预测Anwser是不是一致或者近似一致就好了。

| Question | Answer |
| --- | --- |
| How old  is Biden now | Biden is 81 years old |
| .... | ... |
| .... | .... |

当然这里的问题在于， **我如何廉价地获取类似上面的这种格式的数据集呢**？要知道人工标注是非常昂贵的。   
一个聪明的方案是: 相较于基于问题给出回答， 何不如基于回答来获取问题呢？后者在实现上， 对LLM来说是比较容易的， 所以只要通过LLM来解析文本， 并基于文本内容来由LLM输出对应的问题， 我们就能够获得一个类似于人为标注过的"黄金数据集"(Golden Dataset)

### 具体代码实现--以llamaindex为例子
准备
```python
from llama_index.core import Settings
from llama_index.llms.ollama import Ollama
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

llm = Ollama(model="llama3", request_timeout=60.0, temperature=0.1)
Settings.llm = llm
embed_modle =  HuggingFaceEmbedding("BAAI/bge-base-en-v1.5")
Settings.embed_model = embed_modle
```
> 老样子， 贫穷的我依然选择llama3开局 🤡（理想情况下肯定是上最新版本的chatgpt会更好）

获取数据:
```python
from llama_index.readers.file import PDFReade
from llama_index.core import Document

loader = PDFReader()
docs0 = loader.load_data(file="./data/llama2.pdf")
doc_text = "\n\n".join([d.get_content() for d in docs0])
docs = [Document(text=doc_text)]
```
> llama2.pdf是llama2的论文，下载地址：[https://arxiv.org/pdf/2307.09288](https://arxiv.org/pdf/2307.09288)

```python
from llama_index.core.node_parser import SentenceSplitter

node_parser = SentenceSplitter(chunk_size=1024)
base_nodes = node_parser.get_nodes_from_documents(docs)

```
> 这里我们对文档按句子进行切分


```python
from llama_index.core import VectorStoreIndex

index = VectorStoreIndex(base_nodes)

query_engine = index.as_query_engine(similarity_top_k=2)
```

生成模拟的"黄金数据集"
```python

import nest_asyncio

nest_asyncio.apply()

dataset_generator = DatasetGenerator(
    base_nodes[:20],
    show_progress=True,
    num_questions_per_chunk=3,
)


eval_dataset = await dataset_generator.agenerate_dataset_from_nodes(num=60)
```
> nest_asyncio.apply()是为了方便我们在jupyterlab中运行异步代码， 上面的例子其实不需要（因为jupyterlab本身就是运行在一个eventloop上的）， 但是后面会用到

`eval_dataset`就是我们通过LLM获取的数据集， 如果你还是很好奇DatasetGenerator实现原理， 你只要看一下它的prompt就知道了：
> 如果你不感兴趣的话， 这部分可以跳过

首先我们准备一个可以优美展示prompt的辅助函数：
```python
from IPython.display import Markdown, display

def display_prompt_dict(prompts_dict):
    for k, p in prompts_dict.items():
        text_md = f"**Prompt Key**: {k}<br>" f"**Text:** <br>"
        display(Markdown(text_md))
        print(p.get_template())
        display(Markdown("<br><br>"))
```
然后查看dataset_generator的prompt:
```python
display_prompt_dict(dataset_generator.get_prompts())
```
输出：
```
Context information is below.
---------------------
{context_str}
---------------------
Given the context information and not prior knowledge.
generate only questions based on the below query.
{query_str}
```
See？我们其实是让大模型根据文本来生成问题

## 使用Meta Prompt来调优prompt
首先这个idea来自于这篇论文：[https://arxiv.org/pdf/2309.03409](https://arxiv.org/pdf/2309.03409)  
至于实现， 简单来讲， 就是通过迭代的方式， 让Meta Prompt(元提示)基于现有的prompt表现(是一个随着迭代不断扩展的列表， 它会成为Meta Prompt的一部分来做为上下文的一部分)， 来生成更好的提示来改善我们的初始prompt  
首先我们来看一下我们的元提示长什么样：  
```python
meta_tmpl_str = """\
Your task is to generate the instruction <INS>. Below are some previous instructions with their scores.
The score ranges from 1 to 5.

{prev_instruction_score_pairs}

Below we show the task. The <INS> tag is prepended to the below prompt template, e.g. as follows:

######
<INS>
{prompt_tmpl_str}
#######

The prompt template contains template variables. Given an input set of template variables, the formatted prompt is then given to an LLM to get an output.

Some examples of template variable inputs and expected outputs are given below to illustrate the task. **NOTE**: These do NOT represent the \
entire evaluation dataset.

{qa_pairs_str}

We run every input in an evaluation dataset through an LLM. If the LLM-generated output doesn't match the expected output, we mark it as wrong (score 0).
A correct answer has a score of 1. The final "score" for an instruction is the average of scores across an evaluation dataset.
Write your new instruction (<INS>) that is different from the old ones and has a score as high as possible.

Instruction (<INS>): \
"""
```
这一长串的promt的最终目的是为了生成一个promt的前缀（也就是上面元提示中的Instruction (<INS>)）， 后续会不断地根据生成的Instruction及相应的表现（这个表现就是基于前面我们模拟生成的那个数据集来的）

首先我们需要准备好用来评估LLM问答效果的函数：
```python
from llama_index.core.evaluation.eval_utils import get_responses 
from llama_index.core.evaluation import CorrectnessEvaluator, BatchEvalRunner

evaluator_c = CorrectnessEvaluator()
evaluator_dict = {
    "correctness": evaluator_c,
}
batch_runner = BatchEvalRunner(evaluator_dict, workers=2, show_progress=True)


async def get_correctness(query_engine, eval_qa_pairs, batch_runner):
    eval_qs = [q for q, _ in eval_qa_pairs]
    eval_answers = [a for _, a in eval_qa_pairs]
    pred_responses = get_responses(eval_qs, query_engine, show_progress=True)

    eval_results = await batch_runner.aevaluate_responses(
        eval_qs, responses=pred_responses, reference=eval_answers
    )
    avg_correctness = np.array(
        [r.score for r in eval_results["correctness"]]
    ).mean()
    return avg_correctness
```
可以看到`get_correctness`会通过新的query_engine(内嵌了新生成的提示前缀, 也就是Instruction)来对我们的模拟数据中的问题进行回答， 然后基于这个预测回答和模拟数据中的回答来获得准确率
```python
from llama_index.core import PromptTemplate

QA_PROMPT_KEY = "response_synthesizer:text_qa_template"

meta_tmpl = PromptTemplate(meta_tmpl_str)
```
```python
def format_meta_tmpl(
    prev_instr_score_pairs,
    prompt_tmpl_str,
    qa_pairs,
    meta_tmpl,
):
    """Call meta-prompt to generate new instruction."""
    # format prev instruction score pairs.
    pair_str_list = [
        f"Instruction ():\n{instr}\nScore:\n{score}"
        for instr, score in prev_instr_score_pairs
    ]
    full_instr_pair_str = "\n\n".join(pair_str_list)

    # now show QA pairs with ground-truth answers
    qa_str_list = [
        f"query_str:\n{query_str}\nAnswer:\n{answer}"
        for query_str, answer in qa_pairs
    ]
    full_qa_pair_str = "\n\n".join(qa_str_list)

    fmt_meta_tmpl = meta_tmpl.format(
        prev_instruction_score_pairs=full_instr_pair_str,
        prompt_tmpl_str=prompt_tmpl_str,
        qa_pairs_str=full_qa_pair_str,
    )
    return fmt_meta_tmpl
```
`prev_instr_score_pairs`是之前迭代产出的提示前缀以及相应的评分， 所以它会随着迭代的增加而不断扩展， 它的长度会和迭代次数对应的
```python
def get_full_prompt_template(cur_instr: str, prompt_tmpl):
    tmpl_str = prompt_tmpl.get_template()
    new_tmpl_str = cur_instr + "\n" + tmpl_str
    new_tmpl = PromptTemplate(new_tmpl_str)
    return new_tmpl
```
`get_full_prompt_template`其实就是把我们生成的Intruction和原有的提示进行了拼接， 后续会拿这个拼接完的提示去更新Intruction  

最后就是我们的主流程代码：  
```python
def _parse_meta_response(meta_response: str):
    return str(meta_response).split("\n")[0]


async def optimize_prompts(
    query_engine,
    initial_instr: str,
    base_prompt_tmpl,
    meta_tmpl,
    meta_llm,
    batch_eval_runner,
    eval_qa_pairs,
    exemplar_qa_pairs,
    num_iterations: int = 5,
):
    prev_instr_score_pairs = []
    base_prompt_tmpl_str = base_prompt_tmpl.get_template()

    cur_instr = initial_instr
    for idx in range(num_iterations):
        if idx > 0:
            # first generate
            fmt_meta_tmpl = format_meta_tmpl(
                prev_instr_score_pairs,
                base_prompt_tmpl_str,
                exemplar_qa_pairs,
                meta_tmpl,
            )
            meta_response = meta_llm.complete(fmt_meta_tmpl)
            print(fmt_meta_tmpl)
            print(str(meta_response))
            # Parse meta response
            cur_instr = _parse_meta_response(meta_response)

        # append instruction to template
        new_prompt_tmpl = get_full_prompt_template(cur_instr, base_prompt_tmpl)
        query_engine.update_prompts({QA_PROMPT_KEY: new_prompt_tmpl})

        avg_correctness = await get_correctness(
            query_engine, eval_qa_pairs, batch_runner
        )
        prev_instr_score_pairs.append((cur_instr, avg_correctness))

    # find the instruction with the highest score
    max_instr_score_pair = max(
        prev_instr_score_pairs, key=lambda item: item[1]
    )

    # return the instruction
    return max_instr_score_pair[0], prev_instr_score_pairs
```

然后正式地开始迭代
```python
new_instr, prev_instr_score_pairs = await optimize_prompts(
    query_engine,
    initial_instr,
    base_qa_prompt,
    meta_tmpl,
    llm,  # note: treat llm as meta_llm
    batch_runner,
    eval_qr_pairs,
    exemplar_qr_pairs,
    num_iterations=5,
)


new_qa_prompt = query_engine.get_prompts()[QA_PROMPT_KEY]
print(new_qa_prompt)
```
最后输出的new_qa_prompt应该就是评分最好的那个prompt了， 当然我们也可以打印prev_instr_score_pairs，看看它是不是最好的prompt

## 参考
[https://docs.llamaindex.ai/en/stable/examples/prompts/prompt_optimization](https://docs.llamaindex.ai/en/stable/examples/prompts/prompt_optimization/)



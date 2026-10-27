# RAG增强检索(切分板块)

## 一、文档分块

### 什么是文档分块

文档分块（Chunking）是指把一篇长文档拆分成多个较小且相对完整的文本块（Chunk），再分别进行向量化索引和检索。在RAG系统中，分块连接了文档解析、向量检索和大模型生成，是影响召回质量的关键环节，一个理想的chunk应同时满足以下要求：

> 长度不超过嵌入模型和大语言模型的输入限制；
>
> 块内尽量只表达一个完整、集中的主题；
>
> 脱离原文后仍具有足够的上下文，可以被独立理解；
>
> 保留标题路径、页码、来源和父文档ID等元数据，便于追溯原文。

### 为什么需要分块

第一，**满足大语言模型的上下文窗口限制**。直接输入整篇长文档可能超过模型的最大上下文长度。分块后只把与问题相关的内容送入模型，可以控制上下文规模。
第二，**提高生成上下文的质量**。召回内容过多、过长会增加Token成本，并产生"Lost in the Middle"问题，使真正有用的信息被大量无关内容掩盖。
第三，**满足嵌入模型的输入限制**。嵌入模型通常存在最大Token数，超出后可能被截断。例如部分BGE或 Sentence-Transformers模型的上限为 512 Tokens，过长文本会造成后半部分语义丢失。
第四，**提高检索精度**。一个文本块如果同时包含多个主题，其向量表示会被不同语义平均和稀释，用户查询某个细节时不容易准确召回。较小且主题集中的文本块通常更容易与查询匹配。

![](C:\Users\9331x\AppData\Roaming\Typora\typora-user-images\image-20261027190220624.png)

总结：我们将一个文档通过不同的颜色切分成多个小块，以块为力度进行检索，召回通过不同的算法来拆分成不同的小块的过程就叫分块。

## 二、常见的分块策略

### 1、固定长度分块

**固定长度分块**按照固定字符数或Token数切分文档，并可设置 Overlap ，使相邻块保留一部分重复内容 LlamaIndex 中可以使用 `TokenTextSplitter` ：

```python
from 11ama_index.core.node_parser import TokenTextSplitter 

splitter = TokenTextsplitter(
	chunk_size=500, 
    chunk_overlap=50,
nodes = splitter.get_nodes_from_documents(documents）
```

例如，将一篇1500Tokens的文档按照500Tokens切分：

```
输入文档：
[1 ..................................................1500 tokens]

输出：
Chunk 1:[1---------500]
Chunk 2:               [501-------1000]
Chunk 3:							   [1001--------1500]
```

为了避免关键信息恰好落在两个块的边界，可以让相邻块保留部分重复内容

```
Chunk 1:[--------------]
Chunk 2:            [-------------]
Chunk 3:					    [--------------]
```

**优点**：实现简单、速度快、块大小稳定，便于控制索引规模和成本，适合作为快速验证RAG流程的基线。

**缺点**：可能从句子或段落中间切断，不理解文档结构和语义；Overlap过大还会导致索引冗余和重复召回。

**适用场景**：文本结构不明显、需要快速建立基线、对切分边界要求不高的场景。

### 2、递归分块

递归分块按照一组从粗到细的分隔符逐级切分。它通常先尝试段落，再尝试句子、标点、空格，最后使用字符或  Token 强制切分作为兜底。

```
段落 → 换行 → 句子 → 标点 → 空格 → 字符或Token
```

LlamaIndex 中可以使用 SentenceSplitter：

```python
from llama_index.core.node_parser import Sentencesplitter 

splitter = Sentencesplitter(
	chunk_size=500,
    chunk_overlap=50,
nodes = splitter.get_nodes_from_documents(documents)
```

LangChain中的对应工具是RecursiveCharacterTextSplitter。两者名称不同，但都具有从自然边界到细粒度边界逐级切分的思想。

**递归分块**的完整过程如下：

> 1、先按照段落切分：
>
> 2、检查每个片段是否超过 chunk_size
>
> 3、如果某个段落仍然太长，再按照句子切分；
>
> 4、如果句子仍然太长，再按照标点、空格等更细的边界切分；
>
> 5、最后使用字符或 Token 强制切分作为兜底：
>
> 6、将较小片段重新合并成接近目标长度的Chunk，并保留适量Overlap。

```
输入：一篇长文档
		↓按段落切
	[段落A］[超长段落B］［段落c]
				↓按句子继续切
			[句1][句2］[超长句3]
						↓按标点或字符切
					[片段3.1］[片段3.2]
```

递归分块器本身通常不真正识别标题。标题层级需要由前面的文档解析器提取，再作为结构边界或元数据传给分块器。如果输入没有段落换行和正常标点，递归分块仍能运行，但会逐步退化为字符或Token级强制切分，只能保证长度不超限，不能保证语义边界正确。
**优点**：尽量保留段落和句子的完整性；速度快，不需要调用模型；是普通RAG项目中最常用的分块方式，
**缺点**：依赖文本解析和清洗质量；OCR错误换行可能造成误切；它并不真正理解标题和主题变化，输入没有正常换行或标点时会退化为固定长度切分。
**适用场景**：普通文章、说明书、制度和技术文档，以及已经完成基本清洗的非结构化文本。

![](C:\Users\9331x\AppData\Roaming\Typora\typora-user-images\image-20261027192355004.png)

### 3、结构化分块

**结构化分块**利用文件本身的组织结构确定边界，而不是只根据长度切分。

* Markdown：按照标题、段落和代码块切分；
* HTML：按照h1~h6、p、1i、table等标签切分；
* Docx：按照Heading样式、段落、列表和表格切分；
* 代码：按照文件、类、函数和方法切分：
* PDF/PPT：按照标题、正文、列表、表格、图片和页面区域切分。

 LlamaIndex 提供了 `MarkdownNodeParser` 、`HTMLNodeParser`、`JSONNodeParser`、
`codeSplitter`等解析器。例如：

``` python
from llama_index.core.node_parser import MarkdownNodeParser

parser = MarkdownNodeParser()
nodes = parser.get_nodes_from_documents(documents)
```

结构化分块的典型处理流程如下：

```
原始文件
→文档解析或版面分析 
→识别标题、正文、表格、列表和代码等结构单元 
→在超长结构单元内部继续递归切分
→生成 Chunks ，并保存标题路径、页码和来源等元数据 
```

Reader 负责读取和提取文件内容，NodeParser 负责形成最终用于索引的节点。对于复杂PDF，仅提取纯文本通常无法保留正确阅读顺序，需要结合字号、坐标、页面区域和版面模型识别标题、分栏、表格与图片；扫描PDF还需要先进行 OCR
如果一个章节仍然过长，可以先按标题进行结构化分块，再在章节内部使用**递归分块**。因此，结构化分块和**递归分块**经常组合使用。
**优点**：能够保留标题与正文、表头与数据、函数与代码之间的关系，结果更容易引用和追溯。
**缺点**：不同文件需要不同解析器：复杂PDF需要根据字号、坐标和版面推断结构：解析错误会直接影响后续分块质量。
**适用场景**：Markdown、HTML、Word、代码，以及包含标题、列表和表格的PDF/PPT。

### 4、语义分块

**语义分块**根据主题或语义变化确定边界。常见实现会先切句，再使用嵌入模型计算相邻句子或**句子窗口**的语义距离，在语义突变的位置设置断点，并将连续且语义相近的句子合并成一个Chunk。

LlamaIndex中可以使用`SemanticSplitterNodeParser`：

```python
from llama_index.core.node_parser import SemanticSplitterNodeParser 

splitter = SemanticSplitterNodeParser(
	embed_model=embed_model,
	breakpoint_percentile_threshold=85,
)
nodes = splitter.get_nodes_from_documents(documents)
```

阔值需要根据真实语料调整，数值越低通常越容易产生更多分块。**语义分块**仍需设置最大Token上限，避免产生过长的Chunk。

基于Embedding的语义分块通常包含以下步骤： 

> 1，先把文档切成句子
>
> 2，为单个句子或**句子窗口**生成向量；
>
> 3，计算相邻句子的语义相似度或距离；
>
> 4，在语义差异明显的位置设置断点；
>
> 5，将语义相近的连续句子合并为一个Chunk。
>
> 6，使用最大Token数限制Chunk上限。

```
句1--相似--句2--相似--句3[语义突变]句4-—相似--句5

输出：
Chunk1：句1+句2+句3 
Chunk2：句4+句5
```

语义分块比**递归分块**慢，是因为递归分块主要执行字符串操作，而语义分块需要对大量句子运行嵌入模型、计算相似度，有些算法还需要进行多轮合并。
**优点**：块内主题更一致，不完全依赖换行和格式，能够缓解语义在错误位置被切断的问题。
**缺点**：需要对大量句子计算Embedding和相似度，建库速度较慢：闻值需要实验调整；块大小不固定。
**适用场景**：会议纪要、访谈、长篇报告、OCR后段落结构不可靠的文本，以及一个自然段中经常包含多个主题的内容。

### 5、命题分块

**命题分块**使用大语言模型把原文转换成多条能够独立理解的事实或观点。它可能拆开复合句、补全省略主语、消解代词指代，并进行少量改写，因此不仅文本切割，也属于内容转换和索引增强。

LlamaIndex没有像`SentenceSplitter`一样通用的内置命题分块器。因此，命题提取逻辑需要开发者自已实现，LlamaIndex主要负责承载节点和建立索引。通常需要自行组合以下流程：

```
使用SentenceSplitter粗切原文
→ 调用LLM提取命题列表
→ 每条命题创建一个TextNode
→ 在 metadata 中保存parent_id、页码和来源
→ 使用VectorStoreIndex建立索引
```

下面是命题分块的流程伪代码。它用于说明职责划分，并不是可以直接运行的LlamaIndex内置API:

```python
parent_chunks = sentence_splitter.get_nodes_from_documents(documents) proposition_nodes []

for parent_chunk in parent_chunks :
          # 核心步骤需要自已实现：调用LLM，把原始段落拆成命题列表
		propositions = extract_propositions_with_llm(parent_chunk.text) 
        
        for proposition in propositions:
			proposition_node= create_1lamaindex_node(
					text=proposition,
					parent_node=parent_chunk,
            )
			proposition_nodes.append(proposition_node)
            
index = VectorstoreIndex(proposition_nodes)
```

这里的`extract_propositions_with_llm(）`和`create_llamaindex_node（）`都是为了说明流程而写的自定义函数名，不是 LlamaIndex 自带方法。前者负责通过 Prompt 和 LLM 提取命题，后者负责把命题包装为 LlamaIndex 节点。

其中，"把已经提取出的命题包装成节点"可以使用LlamaIndex自带的TextNode:

```python
from llama_index.core.schema import TextNode 

nodes=[
    TextNode(
        text=proposition, 
        metadata={
            "parent_id"：parent_id,
            "source"source,
            "page":page,
        }，
    )
    for proposition in propositions
]
```

这段TextNode代码本身不负责切分，也不会自动生成命题。它成立的前提是业务代码已经完成以下工作

>a：调用LLM分析原始段落；
>
>b：得到`propositions`命题列表；
>
>c：记录每组命题对应的原始段落ID：
>
>d：再把每条命题保存为`TextNode`。

`text`和`metadata`是`TextNode`提供的字段；`parent_id`、`source`和`page`是项目自定义的
metadata 键，并不是LlamaIndex自动生成的属性。如果需要使用LlamaIndex原生的父子节点关系，也可以通过节点的`relationships`字段保存父节点关系。

例如，原文为：

```
张三于2024年加入公司，并担任技术负责人
他此前在A公司工作了五年。
```

经过命题转换后可以得到：

```
1.张三于2024年加入当前公司。
2.张三担任当前公司的技术负责人。
3.张三此前在A公司工作了五年。
```

**优点**：采用这种策略的主要好处是**事实召回精确**。因为每个检索单元只包含一个单一事实，当用户查询具体信息（如“Vision Pro 定价多少”）时，系统能精准匹配到“定价 3499 美元”这一条命题，而不会召回包含无关信息的整段文本，从而减少噪音，提升回答的准确性。
**缺点**：可能拆散原本的复杂逻辑关系，分块数量也会相应大幅增加。
**适用场景**：这种分块策略通常适用于对**事实精确性**要求极高、且文档内容包含大量独立事实的场景，例如法律条文检索、医疗知识库问答、技术手册或 API 文档查询等。如果你的 RAG 应用更侧重于理解叙事逻辑或复杂关系，则需要谨慎评估是否采用此策略。

## 三、RAG文档分块选型方法论

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/RAG/RAG3-1.png)
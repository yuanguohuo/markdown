---
title: LLM中的名词概念
date: 2026-01-12 20:32:15
tags: [llm,ai]
categories: llm 
---

LLM中的一些名词、概念及其浅析。

<!-- more -->

# Autoencoder 与 Transformer 的关系

## Autoencoder（自编码器）

本质上是一种通用的框架或设计模式，诞生于1987年。它是一种自监督(Self-Supervised Learning)学习策略，无需人工标注标签。其核心思想是通过**编码器-解码器**结构实现数据的压缩与重建，即试图将输入数据压缩到一个低维空间，然后再尽可能完美地还原回来。主要用于降维、去噪、异常检测。关键在于两点：

- 编码器：将输入压缩为低维潜在表示（Latent Representation）；
- 解码器：从潜在表示中尽可能还原原始输入；

具体的编码器和解码器可以用不同的神经网络模块实现，包括 CNN、RNN；也包括transformer。


## Transformer

是一种基于**自注意力机制（Self-Attention）**的神经网络架构，最初由 Google 在 2017 年的论文《Attention Is All You Need》中提出。其核心思想是直接建模输入序列中所有元素（如单词、图像块）之间的全局依赖关系，摒弃传统的循环（RNN）或卷积（CNN）结构。关键优势：1. 并行计算（比 RNN 更快）；2. 捕捉长距离依赖（不受序列长度限制）。

Transformer 也由 **编码器（Encoder）** 和 **解码器（Decoder）** 组成，但可根据任务调整（如 BERT 仅用编码器，GPT 仅用解码器）。

**编码器（Encoder）**

- 输入：序列数据（如句子中的词向量，图像中的分块）。
- 处理流程（每个编码器层重复以下步骤）：
    - 自注意力层（Self-Attention）：计算序列中每个元素与其他元素的关系权重。
    - 前馈神经网络（Feed-Forward Network）：对每个元素独立进行非线性变换。
    - 残差连接（Residual Connection） + 层归一化（LayerNorm）：提升训练稳定性。

**解码器（Decoder）**

- 输入：编码器的输出 + 目标序列（如翻译结果的过去词）。
- 处理流程（每个解码器层）：
    - 掩码自注意力层（Masked Self-Attention）：仅关注当前位置之前的元素（防止未来信息泄漏）。
    - 编码器-解码器注意力层（也叫交叉注意力层, Cross-Attention）：将解码器的中间表示与编码器输出对齐。
    - 前馈神经网络 + 残差连接 + 层归一化：同编码器。

## 结论

根据语境不同，Autoencoder有两种意思：

- 一种通用的框架或设计模式：
- 这种设计模式的一个具体实现；可能是传统的基于 CNN、RNN 的实现，也可能是基于transformer的实现。


# Embedding

## embedding

把 embedding 理解为 token 的数字表示（一个多维向量）有点过于简单。实际上稍微复杂一点。

确实有这种简单的 embedding，可以下载并直接使用。比如 Google 发布的基于 Google News 数据训练的 300 维词向量 (一个词对应一个300维的向量) **word2vec**。值得一提的是：Word2Vec 本身是一种模型架构（如 Skip-Gram 或 CBOW），主要根据海量学习资料中词的相邻关系，来学习词向量（embedding）。实际应用中，通常指代的是通过 Word2Vec 训练得到的预训练词向量文件，如前所说Google发布的300维词向量。这种Word2Vec有一个问题，就是一个单词的 embedding 是静态的，无论上下文是什么，都是相同的，例如 bank 的固定 embedding 向量 `A`（无法区分是银行还是河岸）。

而现代LLM中，引入注意力机制（特别是transformer架构中的自注意力机制）后，一个单词会根据其所在的上下文（Context）拥有多个或动态变化的 Embedding 表示。过程是这样的：

- 初始静态embedding：token的初始嵌入向量，作为推理的输入。
- 计算相关性 (Attention Score)：模型会计算该单词与句子中所有其他单词的相关性（注意力分数）。
- 加权聚合：模型根据这些相关性，从其他单词那里“收集”信息来丰富自己的向量。
- 多层深化：经过多层 transformer 块后，单词的最终向量已经深度融合了上下文信息。单词的最终向量，也叫做最终的动态embedding，或 contextualized representations  或 hidden states. 它不是“embedding layer”的产物，在严谨讨论中，会避免把 hidden states 叫做 “embedding”。但《图解大模型》一书中，把hidden states看成embedding，用于区别word2vec这样的静态embedding，其实word2vec和LLM的初始静态embedding很像，都没有上下文信息（不能区分bank是指银行还是河岸）

这里有几个问题，逐一理解。

1. 从初始静态embedding到最终动态embedding的过程，是推理的一部分吗？还是推理的准备工作？

是推理的一部分；核心正是依赖 多层transformer/self-attention 机制（以及前馈神经网络）。这是现代 transformer 架构大语言模型（LLM）实现“上下文感知”表示的关键。

2. 得到动态embedding之后，下一步是什么？换言之，动态embedding是谁的输入？

最终的动态 embedding（即最后一层 Transformer 输出的 hidden states）会作为「语言模型头（Language Model Head）」的输入，用于预测下一个 token 的概率分布。

3. 初始静态Embedding从哪里来？

简言之：是训练得到的。训练完成后会得到一个**初始嵌入矩阵**或者**Token Embedding Table**。在模型训练开始时随机初始化的一组可学习参数，并在整个预训练过程中通过反向传播不断优化更新。训练结束之后，**初始嵌入表**就是固定的。详细过程：

- 构建词汇表（Vocabulary）
    - 使用 tokenizer（如 BPE、WordPiece、SentencePiece）对大规模语料进行分词。
    - 统计高频子词（subword units），形成固定大小的词汇表（如 GPT-2 用 50257，Llama 用 32000）。
    - 每个子词分配唯一整数 ID（0 到 vocab_size−1）。

- 随机初始化**初始嵌入表**
    - 在模型训练开始前，**初始嵌入表**被随机初始化。
    - 常用方法：从正态分布采样：N(0, σ²)，其中 σ 通常设为 1/√hidden_size（Xavier/Glorot 初始化变种）或均匀分布：U(−√(1/d), √(1/d))，d = hidden_size。目的：打破对称性，让不同 token 初始表示不同，便于梯度下降优化。

- 在预训练中联合学习
    - 嵌入表是模型的可训练参数，与 Transformer 层、LM Head 一起端到端训练。
    - 通过预测下一个 token 的任务（如 causal language modeling），损失函数（如交叉熵）反向传播，不断调整嵌入向量。
    - 结果：语义相近的 token（如 "cat" 和 "dog"）在向量空间中逐渐靠近；语法/语义关系被编码进向量结构中（如 king - man + woman ≈ queen）。

LLM训练结束之后固定下来的参数（或者说 LLM 的可训练参数，模型参数）包括：

- Token Embedding Table，即初始嵌入表
- Transformer层中的权重
    - 注意力机制的参数：如查询（Query）、键（Key）、值（Value）的投影矩阵。
    - 前馈神经网络的参数：各层神经元的权重矩阵和偏置项。
    - 归一化层的参数等。
- 位置编码相关参数（如 RoPE 的 freq 或可学习 pos emb）
- 语言模型头（LM Head）
    - 通常是一个 线性层（Linear Layer）：
    - 所以它包含：权重矩阵W 和 偏置向量 b (有时省略)

推理开始时，通过查**Token Embedding Table**得到各个输入token的embedding。

## 三个阶段

现代大语言模型（LLM）中，token 的向量表示经历了三个阶段性的状态，分别对应训练前、训练后和推理时。

- 训练之前：随机初始化的嵌入向量
    - 存在形式：Token Embedding Table 中的每一行（即每个 token 的初始向量）
    - 特点：从随机分布（如正态分布）采样；无语义信息，仅作为可学习参数的起点；
    - 严格说，这还不是“某个 token 的 embedding”，而是“该 token 对应的参数尚未训练”。

- 训练之后：学到的初始静态 embedding（Input Embedding）
    - 存在形式：训练完成后的 Token Embedding Table
    - 特点：已通过海量文本和语言建模任务优化；同一个 token（如 "apple"）无论出现在什么句子中，查表得到的向量都相同；因此称为 “静态”（static）或 “上下文无关” 的 embedding；注意，还是**上下文无关**
    - 这是推理时输入到 Transformer 的第一层表示；
    - 注意：虽然它“静态”，但已经蕴含丰富的语义和语法结构（比如相似词在向量空间靠近）。

- 推理过程中：上下文相关的动态 embedding（Contextualized Representation）
    - 存在形式：Transformer 最后一层（或某中间层）输出的 hidden states
    - 特点：同一个 token 在不同上下文中产生不同的向量（例："Apple" 在 "I ate an apple." vs "Apple stock rises."）；由 self-attention 机制融合全局上下文生成；称为 “动态” 或 “上下文化” 的 embedding；
    - 这才是 LLM 理解语言的核心表示；
    - 注意：标准 decoder-only LLM（如 GPT、Llama）并不直接输出这种向量作为最终结果——它的目的是预测下一个 token。但你可以从中提取这些 hidden states 用于分析或下游任务。严格说已不是“embedding layer”的产物。


| 阶段                | 向量名称              | 是否可变        | 是否含上下文    | 所属模块                    |
|---------------------|-----------------------|-----------------|-----------------|-----------------------------|
| 训练前              | 随机初始化向量        | 是（待训练）    | 否              | Embedding Table（初始）     |
| 训练后 / 推理输入时 | 初始静态 embedding    | 否（固定）      | 否              | Embedding Table（训练后）   |
| 推理过程中          | 动态上下文 embedding  | 是（随上下文变）| 是              | Transformer 输出            |

## embedding 的歧义

- 在 NLP 领域，“embedding” 有时泛指任何 token/句子的向量表示；但在 LLM 架构中，“embedding layer” 特指输入端的查表操作，其输出是静态的；
- 动态向量通常叫 hidden states、representations 或 contextual embeddings，不属于 embedding layer 的输出。**在严谨讨论中，会避免把 hidden states 叫做 embedding**，但在某些宽松或应用导向的语境中，人们可能会说：“用 LLM 提取句子的 embedding”，这时他们实际指的是从某一层（如最后一层）取出的 hidden states。例如，如下程序：

```
import os
import torch
from transformers import AutoModel, AutoTokenizer

if __name__ == "__main__":
    os.environ["CUDA_VISIBLE_DEVICES"] = ""  # 禁用 GPU

    model_name = "microsoft/deberta-v3-xsmall"

    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModel.from_pretrained(model_name)

    # 编码输入（注意：应传入字典，且 key 是 input_ids 等）
    inputs = tokenizer("Hello, world", return_tensors="pt")

    print(inputs)
    # 打印：{'input_ids': tensor([[5365,  261,  447]]), 'token_type_ids': tensor([[0, 0, 0]]), 'attention_mask': tensor([[1, 1, 1]])}
    # 5365, 261, 447 分别是token '▁Hello', ',', '▁world' 的 ID

    print(tokenizer.convert_ids_to_tokens([5365, 261, 447]))
    # 打印：['▁Hello', ',', '▁world']

    with torch.no_grad():
        static_embeddings = model.embeddings(inputs['input_ids'])

    print(static_embeddings.shape)
    # 打印：[1, 3, 384]

    print(static_embeddings[0, 0, :])  # <-- "Hello" 的初始静态embedding, 384维
    print(static_embeddings[0, 1, :])  # <-- "," 的初始静态embedding, 384维
    print(static_embeddings[0, 2, :])  # <-- "world" 的初始静态embedding, 384维

    # 前向传播
    with torch.no_grad():
        outputs = model(**inputs)

    # 获取最后一层 hidden states: [batch_size, seq_len, hidden_size]
    last_hidden_states = outputs.last_hidden_state

    print(last_hidden_states.shape)
    # 打印：torch.Size([1, 3, 384])

    print(last_hidden_states)
    print(outputs[0])
    # 以上两行打印相同，都是：
    # tensor([[[ 1.1836, -0.1796,  0.4531,  ..., -0.1372,  0.3018,  0.6484],      <-- "Hello" 的上下文表示(动态上下文 embedding)，384维
    #          [-0.2578,  1.7129, -0.4031,  ..., -0.0100, -0.4080,  2.1738],      <-- "," 的上下文表示(动态上下文 embedding)，384维
    #          [-0.2446,  1.3828,  0.5576,  ...,  0.6816, -0.1276,  2.0039]]],    <-- "world" 的上下文表示(动态上下文 embedding)，384维

    print(last_hidden_states[0, 0, :])  # <-- "Hello" 的上下文表示(动态上下文 embedding)，384维 
    print(last_hidden_states[0, 1, :])  # <-- "," 的上下文表示(动态上下文 embedding)，384维     
    print(last_hidden_states[0, 2, :])  # <-- "world" 的上下文表示(动态上下文 embedding)，384维 
```

"microsoft/deberta-v3-xsmall"是一个encoder-only的模型（见下文），它的用途是理解（Understanding），而非生成（Generation），即不能自回归生成文本，只能对输入做编码（得到 embedding；这里的 embedding 指动态上下文 embedding）

# 生成式模型

虽然原始 Transformer 是 encoder-decoder 架构，但在实际应用中，出现了2类变体：

## Encoder-only 模型

如 BERT、RoBERTa，仅使用 Transformer 的 Encoder 部分。

- 使用的是**无掩码自注意力（unmasked self-attention）**
- 每个 token 在计算表示时，可以“看到”整个输入序列的所有其他 token（包括前和后）。即每个 token 的 最终上下文表示（即Contextualized Representation 或者dynamic embedding） 融合了整个输入序列中所有 token 的信息（双向上下文）。
- 这种能力是架构固有的，无论训练还是推理都如此。
- 训练方式：**掩码语言建模（Masked Language Model, MLM）**，即随机**遮盖、替换或者保持**部分 token，让模型预测被遮盖的内容。即随机选择输入序列中 15% 的 token，按一定比例将其替换为 [MASK]、随机词或原词；然后利用双向上下文预测这些被处理过的 token 的原始值，并通过反向传播优化整个模型的参数。
- 适用场景：能利用**双向上下文**信息，对每个 token 的理解更全面。在需要深层语义理解的任务上表现优异，例如分类。
- 局限：不能直接用于生成任务（如文本生成），因为没有自回归机制。

## Decoder-only 模型

如 GPT 系列，仅使用 Transformer 的 Decoder 部分，但去掉交叉注意力模块（因为没有 Encoder）。自回归语言建模（Causal Language Modeling）

- 使用的是**因果掩码自注意力（causal/masked self-attention）**
- 每个 token 只能“看到”它自己及之前的 token。即每个 token 的 最终上下文表示（即Contextualized Representation 或者dynamic embedding）仅基于它自身及左侧 token 的信息。
- 这也是架构决定的，无论训练还是推理都如此。
- 训练方式：模型始终基于左侧已知的上下文预测下一个 token；通过最小化预测与真实 token 之间的损失函数（如交叉熵）；利用反向传播优化所有参数；预测越准确（损失越低），说明模型对语言规律的掌握越好，训练效果越佳。
- 适用场景：天然支持 自回归生成，适合生成连贯文本。架构简单，易于扩展（如 GPT-3、GPT-4）。推理时可逐 token 生成，效率高。
- 局限：只能利用左侧上下文（单向），对当前 token 的理解不如双向模型全面。

## 什么是生成式模型？

两种变体，加上原始transformer，共三种类型。我们常说的生成式模型，就是包含decoder的模型，即decoder-only的和encoder-decoder的模型。

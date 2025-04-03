---
title: Self-Attention 
date: 2025-03-07 20:30:40
tags: [llm,ai]
categories: llm
---

介绍Self-Attention的计算过程。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>


# 输入 (1)

假设当前上下文是“humpty dumpty sat on”，我们要预测下一个单词。有以下输入：

## Eembedding矩阵 (1.1)

假设每个单词的embedding向量是512维，并且假设：

- "humpty"的embedding向量是：

$$
E_1 = [e_{1,1}, e_{1,2}, e_{1,3}, \ldots, e_{1,512}]
$$


- "dumpty"的embedding向量是：

$$
E_2 = [e_{2,1}, e_{2,2}, e_{2,3}, \ldots, e_{2,512}]
$$


- "sat"的embedding向量是：

$$
E_3 = [e_{3,1}, e_{3,2}, e_{3,3}, \ldots, e_{3,512}]
$$

- "on"的embedding向量是：

$$
E_4 = [e_{4,1}, e_{4,2}, e_{4,3}, \ldots, e_{4,512}]
$$


其实就是一个$4 \times 512$的矩阵:

$$
X = \begin{bmatrix}
e_{1,1} & e_{1,2} & e_{1,3} & \cdots & e_{1,512} \\\\
e_{2,1} & e_{2,2} & e_{2,3} & \cdots & e_{2,512} \\\\
e_{3,1} & e_{3,2} & e_{3,3} & \cdots & e_{3,512} \\\\
e_{4,1} & e_{4,2} & e_{4,3} & \cdots & e_{4,512}
\end{bmatrix}
$$

## 投影矩阵 (1.2)

有3个投影矩阵，它们都是$512 \times 512$的矩阵。


$$
W^Q = \begin{bmatrix}
q_{1,1} & q_{1,2} & q_{1,3} & \cdots & q_{1,512} \\\\
q_{2,1} & q_{2,2} & q_{2,3} & \cdots & q_{2,512} \\\\
\vdots & \vdots & \vdots & \ddots & \vdots \\\\
q_{512,1} & q_{512,2} & q_{512,3} & \cdots & q_{512,512}
\end{bmatrix}
$$


$$
W^K = \begin{bmatrix}
k_{1,1} & k_{1,2} & k_{1,3} & \cdots & k_{1,512} \\\\
k_{2,1} & k_{2,2} & k_{2,3} & \cdots & k_{2,512} \\\\
\vdots & \vdots & \vdots & \ddots & \vdots \\\\
k_{512,1} & k_{512,2} & k_{512,3} & \cdots & k_{512,512}
\end{bmatrix}
$$


$$
W^V = \begin{bmatrix}
v_{1,1} & v_{1,2} & v_{1,3} & \cdots & v_{1,512} \\\\
v_{2,1} & v_{2,2} & v_{2,3} & \cdots & v_{2,512} \\\\
\vdots & \vdots & \vdots & \ddots & \vdots \\\\
v_{512,1} & v_{512,2} & v_{512,3} & \cdots & v_{512,512}
\end{bmatrix}
$$


这几个矩阵都是学习得到的（如何学习呢）。**它们叫做投影矩阵，意思是把单词的512维embedding向量投影到512维空间**，见第2节如何使用它们生成每个单词的Query向量、Key向量和Value向量。


# 生成 Query、Key、Value 向量(矩阵) (2)

- "humpty"的Query向量：$Q_{humpty} =$

$$
E_1W^Q = \begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}q_{n,2} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}q_{n,512}
\end{bmatrix}
$$


- "humpty"的Key向量：$K_{humpty} =$

$$
E_1W^K = \begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}k_{n,2} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}k_{n,512}
\end{bmatrix}
$$


- "humpty"的Key向量：$V_{humpty}$

$$
E_1W^V = \begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}v_{n,2} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}v_{n,512}
\end{bmatrix}
$$

同理，我们可以计算出"dumpty", "sat"和"on"的Query，Key，Value向量。

当然，我们也可以使用矩阵表示它们：

- Query矩阵 $Q = XW^Q =$

$$
\begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{1,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}q_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{2,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{2,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{2,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{2,n}q_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{3,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{3,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{3,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{3,n}q_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{4,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{4,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{4,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{4,n}q_{n,512}
\end{bmatrix}
$$

审视一下这个$4 \times 512$的矩阵：首先，第1行是一个512维的向量，每个元素都是第1个单词"humpty"的embedding向量($E_1$)的线性组合，所以，**它是"humpty"的embedding向量的投影**(参考第1.2节的投影矩阵$W^Q$)。或者说，**它是第1个单词"humpty"的一些特征，来自第1个单词的embedding向量**。同理，第2行是第2个单词特征，来自第2个单词的embedding向量；第3行是第3个单词的特征；第4行是第4个单词的特征。

下面的Key矩阵和Value矩阵也是一样，每一行都是一个单词的特征：只是投影矩阵不同($W^K$、$W^V$、$W^Q$互不相同)，得到的特征也不同。

- Key矩阵 $K = XW^K =$

$$
\begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{1,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}k_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{2,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{2,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{2,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{2,n}k_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{3,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{3,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{3,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{3,n}k_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{4,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{4,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{4,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{4,n}k_{n,512}
\end{bmatrix}
$$

- Value矩阵 $V = XW^V =$

$$
\begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{1,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}v_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{2,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{2,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{2,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{2,n}v_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{3,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{3,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{3,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{3,n}v_{n,512} \\\\
\sum\limits_{n=1}^{512}e_{4,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{4,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{4,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{4,n}v_{n,512}
\end{bmatrix}
$$

这些矩阵都是$4 \times 512$的矩阵。它们到底什么意思呢？其实是把每个单词(的embedding向量)投影了3次，得到每个单词的Query投影，Key投影和Value投影。我是这么理解的：

- Value投影代表单词本身(单词的影子)；
- Query投影用于查询自己和别的单词的关系；
- Key投影用于别的单词查询和自己的关系；
- 单词之间的关系就是**单词之间的相关性**，也就是“**注意力分数**”；“注意力分数”经过缩放、归一化处理就是单词之间的权重；
- 用一个单词相对各个单词的权重对各个单词的Value向量进行加权求和，得到一个新的512维向量(因为Value向量是512维，所以新向量也是512维)，就是这个单词的“新的表示”，即这个单词的**Self-Attention-Output**，它包含来自整个序列的信息，帮助模型预测下一个词。加权求和计算，详见第3.3节。

我们使用$Q_x$，$K_x$，$V_x$分别表示单词$x$的Query投影，Key投影和Value投影(它们都是512维的向量)，$x \in [humpty, dumpty, sat, on]$，则：

$$
Q = XW^Q = \begin{bmatrix}
Q_{humpty} \\\\
Q_{dumpty} \\\\
Q_{sat} \\\\
Q_{on}
\end{bmatrix}
$$

$$
K = XW^K = \begin{bmatrix}
K_{humpty} \\\\
K_{dumpty} \\\\
K_{sat} \\\\
K_{on}
\end{bmatrix}
$$

$$
V = XW^V = \begin{bmatrix}
V_{humpty} \\\\
V_{dumpty} \\\\
V_{sat} \\\\
V_{on}
\end{bmatrix}
$$

# 计算一个单词的Self-Attention-Output (3)

以第4个单词"on"为例，计算它的Self-Attention-Output:

## 计算注意力分数 (3.1)

$AttentionScore_{on} =$

$$
\begin{bmatrix}
\frac{Q_{on} \cdot K_{humpty}}{\sqrt{d_k}}, & \frac{Q_{on} \cdot K_{dumpty}}{\sqrt{d_k}}, & \frac{Q_{on} \cdot K_{sat}}{\sqrt{d_k}}, & \frac{Q_{on} \cdot K_{on}}{\sqrt{d_k}}
\end{bmatrix}
$$

投影空间是512维，所以$d_k = 512$。除以$\sqrt{d_k}$用来防止梯度不稳定(vanishing gradient和exploding gradient)，忽略它不影响理解。

看分子，它是"on"的Query向量和其它各个单词的Key向量的**点积**，所以得到的是"on"对于各个单词(包括自己)的注意力分数。


## 计算注意力权重 (3.2)

使用softmax对上一步算出的$AttentionScore_{on}$进行归一化，就的得到"on"相对于其它单词的权重：

$AttentionWeight_{on} = softmax(AttentionScore_{on}) =$

$$
\begin{bmatrix}
w_{on,humpty}, & w_{on,dumpty}, & w_{on,sat}, & w_{on,on}
\end{bmatrix}
$$

对于任意$x$($x \in [humpty, dumpty, sat, on]$)，满足$w_{on,x} \ge 0$并且$\sum\limits_xw_{on,x} = 1$。也就是概率分布。

## 加权聚合 (3.3)

$SelfAttentionOutput_{on} =$

$$
w_{on,humpty}V_{humpty} + w_{on,dumpty}V_{dumpty} + w_{on,sat}V_{sat} + w_{on,on}V_{on}
$$

这是什么意思呢？对于任意$x$($x \in [humpty, dumpty, sat, on]$)，$w_{on,x}$是一个标量，$V_x$是一个512维的向量(即各个单词的Value向量)。所以结果是一个512维的向量。假如权重$w_{on,x}$都为1，那么就是各个单词的Value向量的**各个分量对应相加**：

$$
\begin{bmatrix}
\sum\limits_xV_x分量1, & \sum\limits_xV_x分量2, & \cdots, & \sum\limits_xV_x分量512
\end{bmatrix}
$$

考虑权重，就是各个分量加权相加，即：

$$
\begin{bmatrix}
\sum\limits_xw_{on,x}V_x分量1, & \sum\limits_xw_{on,x}V_x分量2, & \cdots, & \sum\limits_xw_{on,x}V_x分量512
\end{bmatrix}
$$

其中$x \in [humpty, dumpty, sat, on]$。

可见"on"的输入是512维的embedding向量，输出$SelfAttentionOutput_{on}$也是一个512维向量。它是所有单词(包括"on"自己)的Value向量的加权求和。这里的“权”就是“on”和各个单词之间关系的远近。例如：

- $w_{on,humpty} = 0.15$
- $w_{on,dumpty} = 0.25$
- $w_{on,sat} = 0.55$
- $w_{on,on} = 0.05$

注意，“on”和自己的关系也并不近。动词“sat”与介词“on”搭配，所以权重高。所以，$SelfAttentionOutput_{on}$中，来自“sat”的信息最多，使其主导了输出。自注意力机制通过动态计算序列内所有词的相关性，使模型能够捕捉长距离依赖。

单词"on"的embedding向量只是单个词的静态表示，而$SelfAttentionOutput_{on}$向量实际上包含了整个序列中其他词的信息，特别是那些与当前词相关的部分。比如，会更多地受到“sat”的影响，因为它的注意力权重较高。所以，$SelfAttentionOutput_{on}$就不再仅仅是“on”这个词本身的含义，而是结合了它在当前句子中的上下文关系后的动态表示。这种动态表示能够更准确地反映词在特定语境中的意义，从而帮助模型更好地进行后续的预测任务。

# 计算所有单词的SelfAttentionOutput (4)

其实，会算一个单词的Self-Attention-Output也就会算所有单词的；但是可以通过矩阵运算，一次计算所有单词的SelfAttentionOutput。

## 计算注意力分数矩阵 (4.1)

$$
AttentionScore = \frac{QK^T}{\sqrt{d_k}}
$$

这样就一次算出所有单词之间的注意力分数。注意：第3.1节中的**点积**($Q_{on} \cdot K_{humpty}$)，在这里就是矩阵的一行和一列对应相乘再相加。因为$Q$矩阵和$K$矩阵都是按行组织的(一行一个单词，$4 \times 512$)，没法直接相乘，所以$K$需要转置。转置之后就和第3.1节计算**点积**一模一样。

同样，除以$\sqrt{d_k}$用来防止梯度不稳定(vanishing gradient和exploding gradient)。

总之，得到的$AttentionScore$是一个$4 \times 4$的矩阵：第1行是第1个单词“humpty”与各个单词的注意力分数；第2行是第2个单词“dumpty”与各个单词的注意力分数；以此类推。

## 计算注意力权重矩阵 (4.2)

$$
AttentionWeight = softmax(AttentionScore)
$$

注意$softmax$不是针对整个矩阵的，而是逐行归一化，即每行的所有元素独立进行softmax归一化，确保每行的和为1。

将所有元素视为一个向量，整体归一化（所有元素和为1），这种方式几乎不使用：因为这种操作会破坏矩阵的语义结构，导致行/列间依赖关系丢失。

所以，这一步比较简单，没有改变矩阵的结构，$AttentionWeight$还是一个$4 \times 4$的矩阵。就是把$AttentionScore$逐行归一化，转化成概率分布。即$AttentionWeight[m][n]$是单词m对单词n的weight。我们表示为：


$$
AttentionWeight = \begin{bmatrix}
w_{humpty,humpty} & w_{humpty,dumpty} & w_{humpty,sat} & w_{humpty,on} \\\\
w_{dumpty,humpty} & w_{dumpty,dumpty} & w_{dumpty,sat} & w_{dumpty,on} \\\\
w_{sat,humpty}    & w_{sat,dumpty}    & w_{sat,sat}    & w_{sat,on} \\\\
w_{on,humpty}     & w_{on,dumpty}     & w_{on,sat}     & w_{on,on}
\end{bmatrix}
$$

## 加权聚合 (4.3)

$SelfAttentionOutput = AttentionWeight V = $

$$
\begin{bmatrix}
w_{humpty,humpty} & w_{humpty,dumpty} & w_{humpty,sat} & w_{humpty,on} \\\\
w_{dumpty,humpty} & w_{dumpty,dumpty} & w_{dumpty,sat} & w_{dumpty,on} \\\\
w_{sat,humpty}    & w_{sat,dumpty}    & w_{sat,sat}    & w_{sat,on} \\\\
w_{on,humpty}     & w_{on,dumpty}     & w_{on,sat}     & w_{on,on}
\end{bmatrix} V
$$

如第2节所述，$V$可表示为:

$$
V = XW^V = \begin{bmatrix}
V_{humpty} \\\\
V_{dumpty} \\\\
V_{sat} \\\\
V_{on}
\end{bmatrix} = \begin{bmatrix}
V_{humpty}分量1, & V_{humpty}分量2, & \cdots, & V_{humpty}分量512 \\\\
V_{dumpty}分量1, & V_{dumpty}分量2, & \cdots, & V_{dumpty}分量512 \\\\
V_{sat}分量1,    & V_{sat}分量2,    & \cdots, & V_{sat}分量512 \\\\
V_{on}分量1,     & V_{on}分量2,     & \cdots, & V_{on}分量512
\end{bmatrix}
$$

可见，$SelfAttentionOutput$是$4 \times 4$的矩阵乘以$4 \times 512$的矩阵的结果，也就是一个$4 \times 512$的矩阵，和输入矩阵$X$是一致的。

其中第4行，就是第3.3节中的$SelfAttentionOutput_{on}$；同理，第1行是$SelfAttentionOutput_{humpty}$，第2行是$SelfAttentionOutput_{dumpty}$，第3行是$SelfAttentionOutput_{sat}$。以第3行"sat"为例，展开一下：

$SelfAttentionOutput_{sat} = $

$$
\begin{bmatrix}
w_{sat,humpty}, & w_{sat,dumpty}, & w_{sat,sat}, & w_{sat,on}
\end{bmatrix} V
$$

所以，结果是：

$$
\begin{bmatrix}
\sum\limits_xw_{sat,x}V_x分量1, & \sum\limits_xw_{sat,x}V_x分量2, & \cdots, & \sum\limits_xw_{sat,x}V_x分量512
\end{bmatrix}
$$

其中$x \in [humpty, dumpty, sat, on]$。

# 总结 (5)

输出维度与输入一致，本例中输入$X$和输出$SelfAttentionOutput$都是$4 \times 512$的矩阵。对单词$x$而言，输入的是它的512维embedding向量，输出$SelfAttentionOutput_x$也是512维向量，$x \in [humpty, dumpty, sat, on]$。但是，**自注意力输出不是简单的词向量变换，而是一种语境增强的语义表示，为下游任务提供强大的上下文感知能力**。从后文我们将发现，Self Attention是[MultiHead Attention](https://www.yuanguohuo.com/2025/03/08/llm-multi-head-attenion/)的一个特例。

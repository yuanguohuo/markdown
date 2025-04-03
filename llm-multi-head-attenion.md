---
title: MultiHead Attention 
date: 2025-03-08 18:15:30
tags: [llm,ai]
categories: llm
---

介绍MultiHead Attention的计算过程。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 说明 (0)

MultiHead Attention是[Self Attention](https://www.yuanguohuo.com/2025/03/07/llm-self-attention/)的推广，或者说后者是前者的特例。

Self Attention是这样的：

- 把单词的512维的embedding向量在512维空间投影3次，分别生成Query投影、Key投影和Value投影。这些投影都还是512维的，可以理解为没有丢失信息。然后就和单词的embedding向量没有关系了。而是使用这3个投影进行运算。
- 在一个上下文中，使用Query投影和Key投影计算单词两两之间的注意力权重；注意力权重可以理解为单词之间的相关性大小。
- 使用注意力权重加权求和Value投影，得到各个单词的SelfAttentionOutput；SelfAttentionOutput和embedding向量是同构的(都是512维)。

MultiHead Attention：

- 把单词的512维的embedding向量在低维空间(例如64维，可以选择其它维度)投影3次，分别生成Query投影、Key投影和Value投影。这些投影都是64维的，可以理解为丢失了部分信息。然后暂时和单词的embedding向量没有关系了。而是使用这3个投影进行运算。
- 在一个上下文中，使用Query投影和Key投影计算单词两两之间的注意力权重；注意力权重可以理解为单词之间的相关性大小。
- 使用注意力权重加权求和Value投影，得到各个单词的Output；Output和embedding向量不是同构的，embedding是512维，而Output是64维。**不过到这里还没结束**。

到现在为止，除了投影到低维(64维)且生成的Output也是低维的(64维)之外，和Self Attention没有任何区别。但是，MultiHead Attention要**把面的过程重复多次**。重复几次呢？

答案是$\frac{512}{64}=8$次。如此以来，**把8次的结果拼起来，又得到512维的Output，和embedding保持同构！**这也是“multi”的由来。

最后，使用一个$512 \times 512$的线性变换矩阵$W^O$(见1.4节)，对512维的Output进行一个线性变换，就得到最终的MultiHeadOutput！

可见，**如果在选择低维空间进行投影的时候，还选择512维(把512维看作一个特殊的低维)，那么重复的次数就是$\frac{512}{512}=1$次！再选择$512 \times 512$的单位矩阵作为线性变换矩阵$W^O$，MultiHead Attention就退化成Self Attention了**。

我是这么理解的：**从高维(512)到低维(64)投影时丢失了部分信息，但是我们换了8个角度，投了8次(把Query投影-Key投影-Value投影算1次)，这样就尽可能的保留了本体的特征。并且和Self Attention的1次同维投影相比，多角度的低维投影可能更能抓住本体的特征。这里，本体的特征就是单词间的关系！

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


## 头数 (1.2)

从前面的说明可知，选择**头数**和选择**低维空间的维数**是同一个事儿！假设头数为8：

$$ h = 8 $$

那么每个头的维度为：

$$ d_k = \frac{512}{8} = 64 $$

## 投影矩阵 (1.3)

对于每个头$i \in \{1, 2, \ldots, 8\}$，有3个投影矩阵，它们都是$512 \times 64$的矩阵(如第0节所述)。**每个头的投影矩阵不同，当然也不能相同！**否则的话，就是同样的运算重复8次！

$$
W_i^Q = \begin{bmatrix}
q_{1,1} & q_{1,2} & q_{1,3} & \cdots & q_{1,64} \\\\
q_{2,1} & q_{2,2} & q_{2,3} & \cdots & q_{2,64} \\\\
\vdots & \vdots & \vdots & \ddots & \vdots \\\\
q_{512,1} & q_{512,2} & q_{512,3} & \cdots & q_{512,64}
\end{bmatrix}
$$


$$
W_i^K = \begin{bmatrix}
k_{1,1} & k_{1,2} & k_{1,3} & \cdots & k_{1,64} \\\\
k_{2,1} & k_{2,2} & k_{2,3} & \cdots & k_{2,64} \\\\
\vdots & \vdots & \vdots & \ddots & \vdots \\\\
k_{512,1} & k_{512,2} & k_{512,3} & \cdots & k_{512,64}
\end{bmatrix}
$$


$$
W_i^V = \begin{bmatrix}
v_{1,1} & v_{1,2} & v_{1,3} & \cdots & v_{1,64} \\\\
v_{2,1} & v_{2,2} & v_{2,3} & \cdots & v_{2,64} \\\\
\vdots & \vdots & \vdots & \ddots & \vdots \\\\
v_{512,1} & v_{512,2} & v_{512,3} & \cdots & v_{512,64}
\end{bmatrix}
$$


这几个矩阵都可以通过学习得到，其中的意义也不好解释。**但它们叫做投影矩阵，意思是把单词的512维embedding向量投影到64维空间**，见第2节生成Query, Key, Value矩阵。

## 线性变换矩阵 (1.4)

这是个$512 \times 512$的矩阵，也通过学习得到的。在MultiHead Attention输出结果之前，对结果进行一个线性变换。显然，若它是单位矩阵，则相当于没有变换。我们把它记作$W^O$。具体形状不必展开。

# 生成 Query、Key、Value 矩阵 (2)


对于每个头$i \in \{1, 2, \ldots, 8\}$：

- Query矩阵 $Q_i = XW_i^Q =$

$$
\begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{1,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}q_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{2,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{2,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{2,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{2,n}q_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{3,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{3,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{3,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{3,n}q_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{4,n}q_{n,1} & \sum\limits_{n=1}^{512}e_{4,n}q_{n,2} & \sum\limits_{n=1}^{512}e_{4,n}q_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{4,n}q_{n,64}
\end{bmatrix}
$$

审视一下这个矩阵：首先，第1行是一个64维的向量，每个元素都是第1个单词"humpty"的embedding向量($E_1$)的线性组合。注意到embedding向量是512维，而这一行是64维，所以我们说，**它是"humpty"的512维的embedding向量在64维空间的投影**(参考第1.3节的投影矩阵$W_i^Q$)。由于是高维(512)到低维(64)的投影，所以一定会丢失一些信息，就像3维到2维投影只保留了形状而丢失了深度信息。总之，**第1行就是第1个单词"humpty"的一些特征，来自原始的512维的embedding向量**。同理，第2行是第2个单词特征，来自512维的embedding向量；第3行是第3个单词的特征；第4行是第4个单词的特征。

下面的Key矩阵和Value矩阵也是一样，每一行都是一个单词的特征：只是投影矩阵不同($W_i^K$、$W_i^V$、$W_i^Q$互不相同)，得到的特征也不同。

- Key矩阵 $K_i = XW_i^K =$

$$
\begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{1,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}k_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{2,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{2,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{2,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{2,n}k_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{3,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{3,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{3,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{3,n}k_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{4,n}k_{n,1} & \sum\limits_{n=1}^{512}e_{4,n}k_{n,2} & \sum\limits_{n=1}^{512}e_{4,n}k_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{4,n}k_{n,64}
\end{bmatrix}
$$

- Value矩阵 $V_i = XW_i^V =$

$$
\begin{bmatrix}
\sum\limits_{n=1}^{512}e_{1,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{1,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{1,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{1,n}v_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{2,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{2,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{2,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{2,n}v_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{3,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{3,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{3,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{3,n}v_{n,64} \\\\
\sum\limits_{n=1}^{512}e_{4,n}v_{n,1} & \sum\limits_{n=1}^{512}e_{4,n}v_{n,2} & \sum\limits_{n=1}^{512}e_{4,n}v_{n,3} & \cdots & \sum\limits_{n=1}^{512}e_{4,n}v_{n,64}
\end{bmatrix}
$$

到目前，我们只是把每个单词的embedding向量在64维空间投影3次：Query投影，Key投影，Value投影。我是这么理解的：

- Value投影代表embedding本身(自己的影子)；
- Query投影用于查询自己和别的单词的关系；
- Key投影用于别的单词查询和自己的关系；
- 单词之间的关系就是**单词之间的相关性**，也就是“**注意力分数**”；“注意力分数”经过缩放、归一化处理就是单词之间的权重；
- 用一个单词相对各个单词的权重对各个单词的Value向量进行加权求和，得到一个新的64维向量(因为Value向量是64维，所以新向量也是64维)，就是这个头的Output。

我们使用$Q_x$，$K_x$，$V_x$分别表示单词$x$的Query投影，Key投影和Value投影(它们都是64维的向量)，$x \in [humpty, dumpty, sat, on]$，则：

$$
Q_i = XW_i^Q = \begin{bmatrix}
Q_{humpty} \\\\
Q_{dumpty} \\\\
Q_{sat} \\\\
Q_{on}
\end{bmatrix}
$$

$$
K_i = XW_i^K = \begin{bmatrix}
K_{humpty} \\\\
K_{dumpty} \\\\
K_{sat} \\\\
K_{on}
\end{bmatrix}
$$

$$
V_i = XW_i^V = \begin{bmatrix}
V_{humpty} \\\\
V_{dumpty} \\\\
V_{sat} \\\\
V_{on}
\end{bmatrix}
$$

# 计算单头注意力 (3)

以第$i$个头为例：

## 计算注意力分数矩阵 (3.1)

$$
AttentionScore_i = \frac{Q_iK_i^T}{\sqrt{d_k}}
$$

这个矩阵展开就很吓人了。但不要恐慌，分析一下，还是比较简单的。

首先把分母忽略掉，它就是除以标量$\sqrt{d_k} = \sqrt{64} = 8$，不改变矩阵的结构。这是一个缩放因子，用于防止点积结果过大导致梯度消失或爆炸问题。

然后重点看分子部分：它是$Q_i$乘以$K_i$的转置($K_i^T$)。$Q_i$和$K_i$都是$4 \times 64$的矩阵，不转置肯定没法相乘。$K_i$的转置($K_i^T$)就是$64 \times 4$的矩阵，这样以来，结果就是$4 \times 4$的矩阵。

再看怎么相乘的：$K_i$的转置($K_i^T$)，就是把第1行变成第1列，第2行变成第2列，第3行变成第3列……；然后$Q_i$和它相乘。用肉眼看的话，刚好也不用转置了：

- 直接拿$Q_i$的第1行和$K_i$的第1行对应相乘再相加(点积)，得到$AttentionScore_i[1][1]$；即第1个单词的Query投影点乘第1个单词Key投影；
- 直接拿$Q_i$的第1行和$K_i$的第2行对应相乘再相加(点积)，得到$AttentionScore_i[1][2]$；即第1个单词的Query投影点乘第2个单词Key投影；
- 直接拿$Q_i$的第1行和$K_i$的第3行对应相乘再相加(点积)，得到$AttentionScore_i[1][3]$；即第1个单词的Query投影点乘第3个单词Key投影；
- 直接拿$Q_i$的第1行和$K_i$的第4行对应相乘再相加(点积)，得到$AttentionScore_i[1][4]$；即第1个单词的Query投影点乘第4个单词Key投影；

- 直接拿$Q_i$的第2行和$K_i$的第1行对应相乘再相加(点积)，得到$AttentionScore_i[2][1]$；即第2个单词的Query投影点乘第1个单词Key投影；
- 直接拿$Q_i$的第2行和$K_i$的第2行对应相乘再相加(点积)，得到$AttentionScore_i[2][2]$；即第2个单词的Query投影点乘第2个单词Key投影；
- 直接拿$Q_i$的第2行和$K_i$的第3行对应相乘再相加(点积)，得到$AttentionScore_i[2][3]$；即第2个单词的Query投影点乘第3个单词Key投影；
- 直接拿$Q_i$的第2行和$K_i$的第4行对应相乘再相加(点积)，得到$AttentionScore_i[2][4]$；即第2个单词的Query投影点乘第4个单词Key投影；

- ...

前面我们使用$Q_x$，$K_x$分别表示Query投影和Key投影，$x \in [humpty, dumpty, sat, on]$，则：

$AttentionScore_i = $

$$
\begin{bmatrix}
Q_{humpty} \cdot K_{humpty}, & Q_{humpty} \cdot K_{dumpty}, & Q_{humpty} \cdot K_{sat}, & Q_{humpty} \cdot K_{on} \\\\
Q_{dumpty} \cdot K_{humpty}, & Q_{dumpty} \cdot K_{dumpty}, & Q_{dumpty} \cdot K_{sat}, & Q_{dumpty} \cdot K_{on} \\\\
Q_{sat}    \cdot K_{humpty}, & Q_{sat}    \cdot K_{dumpty}, & Q_{sat}    \cdot K_{sat}, & Q_{sat}    \cdot K_{on} \\\\
Q_{on}     \cdot K_{humpty}, & Q_{on}     \cdot K_{dumpty}, & Q_{on}     \cdot K_{sat}, & Q_{on}     \cdot K_{on}
\end{bmatrix}
$$

$AttentionScore_i[m][n]$到底是什么意义呢？它其实表示**单词m对单词n的注意力分数**；

## 应用Softmax归一化 (3.2)

$$
AttentionWeight_i = softmax(AttentionScore_i)
$$

注意$softmax$不是针对整个矩阵的，而是逐行归一化，即每行的所有元素独立进行softmax归一化，确保每行的和为1。

将所有元素视为一个向量，整体归一化（所有元素和为1），这种方式几乎不使用：因为这种操作会破坏矩阵的语义结构，导致行/列间依赖关系丢失。

所以，这一步比较简单，没有改变矩阵的结构，$AttentionWeight_i$还是一个$4 \times 4$的矩阵。就是把$AttentionScore_i$归一化，转化成概率分布。即$AttentionWeight_i[m][n]$是单词m对单词n的weight。我们表示为：


$$
AttentionWeight = \begin{bmatrix}
w_{humpty,humpty} & w_{humpty,dumpty} & w_{humpty,sat} & w_{humpty,on} \\\\
w_{dumpty,humpty} & w_{dumpty,dumpty} & w_{dumpty,sat} & w_{dumpty,on} \\\\
w_{sat,humpty}    & w_{sat,dumpty}    & w_{sat,sat}    & w_{sat,on} \\\\
w_{on,humpty}     & w_{on,dumpty}     & w_{on,sat}     & w_{on,on}
\end{bmatrix}
$$

## 加权聚合Value (3.3)

前面说过，矩阵$V_i$是一个$4 \times 64$的矩阵，也就是每个单词的Value投影，每行是一个单词。每列呢？可以看作是每个单词的一个投影分量，即：

$$
V_i = \begin{bmatrix}
V_{humpty} \\\\
V_{dumpty} \\\\
V_{sat} \\\\
V_{on}
\end{bmatrix} = \begin{bmatrix}
V_{humpty}分量1, & V_{humpty}分量2, & \cdots, & V_{humpty}分量64 \\\\
V_{dumpty}分量1, & V_{dumpty}分量2, & \cdots, & V_{dumpty}分量64 \\\\
V_{sat}分量1,    & V_{sat}分量2,    & \cdots, & V_{sat}分量64 \\\\
V_{on}分量1,     & V_{on}分量2,     & \cdots, & V_{on}分量64
\end{bmatrix}
$$

上一小节我们得到$AttentionWeight_i$是一个$4 \times 4$的矩阵，它乘以$V_i$ ($4 \times 64$)，就得到$4 \times 64$的加权聚合Value：

$Head_i = AttentionWeight_i \cdot V_i = $

$$
\begin{bmatrix}
\sum\limits_{x}w_{humpty,x}V_x分量1, & \sum\limits_{x}w_{humpty,x}V_x分量2, & \cdots, & \sum\limits_{x}w_{humpty,x}V_x分量64 \\\\
\sum\limits_{x}w_{dumpty,x}V_x分量1, & \sum\limits_{x}w_{dumpty,x}V_x分量2, & \cdots, & \sum\limits_{x}w_{dumpty,x}V_x分量64 \\\\
\sum\limits_{x}w_{sat,x}V_x分量1,    & \sum\limits_{x}w_{sat,x}V_x分量2,    & \cdots, & \sum\limits_{x}w_{sat,x}V_x分量64 \\\\
\sum\limits_{x}w_{on,x}V_x分量1,     & \sum\limits_{x}w_{on,x}V_x分量2,     & \cdots, & \sum\limits_{x}w_{on,x}V_x分量64
\end{bmatrix}
$$


- $Head_i[humpty][1]$：所有单词的Value的分量1加权求和；这里的权是指humpty对其它单词的AttentionWeight。
- $Head_i[humpty][2]$：所有单词的Value的**分量2**加权求和；这里的权是指humpty对其它单词的AttentionWeight。
- ...
- $Head_i[sat][1]$：所有单词的Value的分量1加权求和；这里的权是指sat对其它单词的AttentionWeight。
- $Head_i[sat][2]$：所有单词的Value的分量2加权求和；这里的权是指sat对其它单词的AttentionWeight。
- ...

就是说，**第1行是humpty的($Head_i$的)Output；第2行是dumpty的Output；** ……
每个单词的Output又是**所有单词的Value的加权求和**。因为Value是一个向量，所以，**加权求和是指各个分量对应加权求和**！
这里的**权是指当前单词对于所有单词的AttentionWeight**。

总之，$Head_i$是个$4 \times 64$的矩阵，其中第1行中编码了第1个单词与其它单词之间的关联信息；第2行中编码了第2个单词与其它单词的关联信息，……

# 合并多头输出 (4)

到目前为止，我们都在看一个头；得到的$Head_i$也是这一个头$i$的Output。它是一个$4 \times 64$的矩阵。

总共有8个头，重复上面的过程，得到8个$4 \times 64$的Output！拼起来就是$4 \times 512$的矩阵，和输入矩阵$X$同构。

$$
MultiHead(X) = \begin{bmatrix}
Head_1; & Head_2; & Head_3; & \cdots; & Head_8
\end{bmatrix}
$$

其实这8个头，只是输入的投影矩阵不同，其它计算都一模一样。

# 输出线性变换 (5)

上面已经得到和输入$X$的同构矩阵$MultiHead(X)$ ($4 \times 512$)，再乘以线性变换矩阵$W^O$ ($512 \times 512$，见第1.4节)，得到的还是$4 \times 512$的矩阵。即:

$$
MultiHeadOutput = MultiHead(X)W^O \in R^{4 \times 512}
$$

# 总结 (6)

- 输入与输出对齐：输出维度与输入一致，便于残差连接和后续层处理。
- 多头协作：每个头学习不同模式，最终输出融合多视角信息。
- 上下文编码：每个输出位置聚合了全局依赖关系，为预测提供丰富特征。

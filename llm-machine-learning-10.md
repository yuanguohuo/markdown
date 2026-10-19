---
title: 机器学习原理-自注意力
date: 2025-12-08 21:30:20
tags: [llm,attention]
categories: llm
---

自注意力（Self Attention）的计算过程。

说明：数学上，向量 $\boldsymbol{v}$ 通常指列向量（即 $d \times 1$ 的矩阵），其线性投影记作 $\boldsymbol{M} \boldsymbol{v}$，其中 $\boldsymbol{M}$ 是投影矩阵，形状是 $r \times d$。若改用行向量表示，则需转置为 $\boldsymbol{v}^\top \boldsymbol{M}^\top$。而本文遵从 Transformer 的惯例：向量（如 embedding、query、key、value）特指行向量，其中 embedding 向量是 $1 \times d\_{model}$，query 和 key 向量是 $1 \times d\_k$，value 向量是 $1 \times d\_v$（其中 $d\_k$ 和 $d\_v$ 不一定相等）。同时权重矩阵 $\boldsymbol{W}^Q$、$\boldsymbol{W}^K$ 和 $\boldsymbol{W}^V$ 的形状分别直接定义成 $d\_{model} \times d\_k$、$d\_{model} \times d\_k$ 和 $d\_{model} \times d\_v$，使得投影运算可以直接写成 $\boldsymbol{e} \boldsymbol{W}^Q$、$\boldsymbol{e} \boldsymbol{W}^K$、$\boldsymbol{e} \boldsymbol{W}^V$，无需额外转置。这样记的好处是更接近于编程：$n$ 个 token 的 embedding 向量组成 $n \times d\_{model}$ 的矩阵，每行对应一个 token，批量投影只需一次矩阵乘法。

<!-- more -->

# 计算过程 (1)

假设一句话包含 4 个 token，它们的 embedding 分别是 $\boldsymbol{e}\_1$、$\boldsymbol{e}\_2$、$\boldsymbol{e}\_3$ 和 $\boldsymbol{e}\_4$。并假设权重矩阵（$\boldsymbol{W}^Q$、$\boldsymbol{W}^K$ 和 $\boldsymbol{W}^V$）已知。

## 计算 Attention Score (1.1)

计算各个 token 的 key 向量 $\boldsymbol{k}\_i$（$i=1,2,3,4$）。

$$
\begin{aligned}
\boldsymbol{k}\_1 & = \boldsymbol{e}\_1 \boldsymbol{W}^K \\\\
\boldsymbol{k}\_2 & = \boldsymbol{e}\_2 \boldsymbol{W}^K \\\\
\boldsymbol{k}\_3 & = \boldsymbol{e}\_3 \boldsymbol{W}^K \\\\
\boldsymbol{k}\_4 & = \boldsymbol{e}\_4 \boldsymbol{W}^K
\end{aligned}
\tag{1}
$$

Token 的 key 向量定义**我能被什么匹配到**。

然后，以 token $\boldsymbol{e}\_1$ 为例，计算它的 query 向量 $\boldsymbol{q}\_1$：

$$
\boldsymbol{q}\_1 = \boldsymbol{e}\_1 \boldsymbol{W}^Q
\tag{2}
$$

Query 向量定义**我在找什么**。Token $\boldsymbol{e}\_1$ 和 $\boldsymbol{e}\_2$ 的匹配度（关联性）就是注意力得分——即 $\boldsymbol{e}\_1$ 对 $\boldsymbol{e}\_2$ 的注意力，记作 $\alpha\_{12}$。

$$
\alpha\_{12} = \boldsymbol{q}\_1 \cdot \boldsymbol{k}\_2
$$

同理，计算 token $\boldsymbol{e}\_1$ 和 $\boldsymbol{e}\_3$、$\boldsymbol{e}\_4$ 的注意力得分，以及**自己和自己的注意力得分**。汇总如下：

$$
\begin{aligned}
\alpha\_{11} & = \boldsymbol{q}\_1 \cdot \boldsymbol{k}\_1 \\\\
\alpha\_{12} & = \boldsymbol{q}\_1 \cdot \boldsymbol{k}\_2 \\\\
\alpha\_{13} & = \boldsymbol{q}\_1 \cdot \boldsymbol{k}\_3 \\\\
\alpha\_{14} & = \boldsymbol{q}\_1 \cdot \boldsymbol{k}\_4
\end{aligned}
$$

可见，注意力得分就是 query 向量和 key 向量的点积或内积（dot-product 或 inner-product）。

数学上，两个向量的点积是：

$$
\boldsymbol{q}\_i \cdot \boldsymbol{k}\_j = \\lVert \boldsymbol{q}\_i \\rVert \\lVert \boldsymbol{k}\_j \\rVert \cos(\theta)
$$

其中 $\theta$ 是两个向量的夹角。所以，点积就是两个向量的**模长相乘，再乘以夹角的余弦值**。也就是说，在模长相近的情况下，**两个向量方向越一致（夹角越小），注意力得分越高**。换言之，**token 之间的相关性，体现在 query 向量和 key 向量的夹角上**。

假设 query 向量代表“它”，句子中 token 的 key 向量分别代表：

- $\boldsymbol{k}\_1$：行人
- $\boldsymbol{k}\_2$：街道
- $\boldsymbol{k}\_3$：黄狗
- $\boldsymbol{k}\_4$：下雨

经过训练，模型会让“它”的 query 向量和“黄狗”的 key 向量夹角较小，于是：

$$
\begin{aligned}
\boldsymbol{q}\_{\text{它}} \cdot \boldsymbol{k}\_{\text{黄狗}} \gg \boldsymbol{q}\_{\text{它}} \cdot \boldsymbol{k}\_{\text{行人}} \\\\
\boldsymbol{q}\_{\text{它}} \cdot \boldsymbol{k}\_{\text{黄狗}} \gg \boldsymbol{q}\_{\text{它}} \cdot \boldsymbol{k}\_{\text{街道}} \\\\
\boldsymbol{q}\_{\text{它}} \cdot \boldsymbol{k}\_{\text{黄狗}} \gg \boldsymbol{q}\_{\text{它}} \cdot \boldsymbol{k}\_{\text{下雨}}
\end{aligned}
$$

为什么不是直接用原始向量？因为原始 embedding 向量并不适合直接做相似度比较。模型会用 $\boldsymbol{W}^Q$、$\boldsymbol{W}^K$ 和 $\boldsymbol{W}^V$ 把输入投影到三个不同的子空间：

- $Q$ 空间：定义“我在找什么”
- $K$ 空间：定义“我能被什么匹配到”
- $V$ 空间：定义“我能提供什么信息”（见后文）

所以，注意力**不是简单比较两个词像不像**，而是在学出来的语义子空间里比较投影的相似度。

> 补充：点积只是计算 Attention Score 的**一种**方法。实际上，还有别的计算方法，例如 Additive（加性注意力）：$\tanh(\boldsymbol{q} + \boldsymbol{k}) \boldsymbol{W}$，过程是：
> - 把 query 和 key 向量逐元素相加（而不是点乘），结果是一个行向量；
> - 相加后的向量做逐元素 $\tanh$（双曲正切函数），结果仍是一个行向量；
> - 最后经过一个线性层（$\boldsymbol{W}$ 是一个列向量），得到标量 $\alpha$（注意力得分）。

## 模长代表什么 (1.2)

上一节说，**在模长相近的情况下**，两个向量方向越一致（夹角越小），注意力得分越高。换言之，token 之间的相关性，体现在 query 向量和 key 向量的夹角上。可是，**影响点积结果的不止夹角，还有两个向量的模长**。

在训练好的表示空间里，一个向量的模长大，通常意味着这个 token 在该注意力头所关注的维度上更活跃、更突出。比如句子“我去吃饭”中，“我”的向量可能模长最大，因为它作为主语与后面每个词都有关联；而“去”的模长较小，角色相对单一。

点积天然保留了这个信息：

- 若某个 token 的 key 向量模长较大，它会与更多 query 产生较大的点积，从而更容易被其他 token 关注到；
- 若某个 token 的 query 向量模长较大，它对所有 key 的点积都会整体变大，得分之间的绝对差距也随之放大；而 softmax 对数值差距敏感，经过 softmax 后，注意力分布会更尖锐，更集中在得分最高的几个 key 上。注：softmax 在下一节引入。

所以，注意力得分大致可以理解为：

$$
\text{AttentionScore} = \text{双方重要性} \times \text{匹配度}
$$

确实也有些模型会显式做 QK-Norm，例如部分现代 LLM（如 Qwen3）会在计算点积前对每个头的 $\boldsymbol{q}$、$\boldsymbol{k}$ 做 RMSNorm 或 L2 归一化。这就更接近**只看向量的方向（匹配度）**。不过实际实现通常还会乘一个可学习缩放，相当于一个可学温度（用于捕获 token 的重要性）。

## 归一化成注意力权重 (1.3)

第 1.1 节得到的注意力得分 $\alpha\_{1j}$ 只是原始匹配分数，它们可以是任意实数，不能直接当作权重使用。为了让它们变成一组注意力权重（大于0，且和为1），需要对它们做归一化。最常用的归一化方法是 softmax。

这里有一个细节：原始 Transformer 在 softmax 之前还会除以 $\sqrt{d\_k}$（其中 $d\_k$ 是 key 向量的维度）。**注意，这里把缩放后的得分仍记作** $\alpha\_{1j}$，即先计算缩放后的得分，再做 softmax。

$$
\alpha\_{1j} = \frac{\boldsymbol{q}\_1 \cdot \boldsymbol{k}\_j}{\sqrt{d\_k}} \quad \quad j=1,2,3,4
\tag{3}
$$

这一步叫**缩放点积**。缩放不是归一化，缩放之后仍然不是权重——可能小于0，总和也不为1。缩放的作用是**防止** $d\_k$ **较大时点积值过大，导致 softmax 进入梯度极小的饱和区**，影响训练稳定性。

缩放之后，送入 softmax 进行归一化。对于 token $\boldsymbol{e}\_1$，归一化得到：

$$
w\_{1j} = \operatorname{softmax}\left(\alpha\_{1j}\right) = \frac{\exp(\alpha\_{1j})}{\sum\_{t=1}^4 \exp(\alpha\_{1t})} \quad \quad j=1,2,3,4
$$

这样得到的 $w\_{1j}$ 都为正且和为 1，可以解释为权重，即**注意力权重（Attention Weight）**。或者说，$w\_{1j}$ 表示 token $\boldsymbol{e}\_1$ **在聚合上下文信息时，分配给** token $\boldsymbol{e}\_j$ **的注意力的多少**。

总结起来，注意力权重（Attention Weight）计算过程如下：

$$
w\_{1j} = \operatorname{softmax}\left(\frac{\boldsymbol{q}\_1 \cdot \boldsymbol{k}\_j}{\sqrt{d\_k}}\right) \quad \quad j=1,2,3,4
$$

也可以通过下图表示：

![figure1](attention-weight.png)
<div style="text-align: center;"><em>Attention Weight</em></div>

说明：在[分类生成式模型](https://www.yuanguohuo.com/2025/11/20/llm-machine-learning-3/)的第 6.4 节，我们也曾采用 softmax 进行归一化，那里用 softmax 有其背后的道理；相比之下，这里就没有那么多道理可言，单纯为了归一化成**权重**。也可以不用 softmax，有人尝试过使用 ReLU，效果也不错。

## 使用注意力权重组合 value 向量 (1.4)

第 1.1 节说过，value 向量定义**我能提供什么信息**：每个 token 的 embedding 向量经过 $\boldsymbol{W}^V$ 投影，得到它的 value 向量：

$$
\boldsymbol{v}\_i = \boldsymbol{e}\_i \boldsymbol{W}^V \quad \quad i=1,2,3,4
\tag{4}
$$

每个 $\boldsymbol{v}\_i$ 是 $1 \times d\_v$ 的行向量。文首说明已声明 $d\_v$ 和 $d\_k$ 不一定相等：key 向量负责“被匹配”，value 向量负责“提供信息”，两件事不必共用同一个维度。

有了注意力权重，token $\boldsymbol{e}\_1$ 就把全句 token 的 value 向量组合起来：

$$
\boldsymbol{o}\_1 = \sum\_{j=1}^4 w\_{1j} \boldsymbol{v}\_j
\tag{5}
$$

即**按注意力权重对 value 向量加权求和**，得到 $\boldsymbol{o}\_1$（同样是 $1 \times d\_v$ 的行向量），也就是 token $\boldsymbol{e}\_1$ 的自注意力输出。整个过程也可以通过下图表示：

![figure2](self-attention-flow.png)
<div style="text-align: center;"><em>Self Attention</em></div>

显然，token $\boldsymbol{e}\_1$ 的自注意力输出 $\boldsymbol{o}\_1$ **包含了所有 token 的信息**——所有 token 的 value 向量（value 向量提供信息）。**更重要的是，若 token** $\boldsymbol{e}\_1$ **和** $\boldsymbol{e}\_j$ **相关性大，则权重** $w\_1j$ **就大，其它权重就小（因为它们的和为 1），最终导致** $\boldsymbol{o}\_1$ **包含** $\boldsymbol{v}\_j$ **的信息更多。**

接前文的例子：若训练后“它”对“黄狗”的注意力权重接近 1、对其余 token 的权重接近 0，则 $\boldsymbol{o}\_{\text{它}}$ 几乎等于 $\boldsymbol{v}\_{\text{黄狗}}$——“黄狗”提供给上下文的信息，几乎原封不动地流进了“它”的输出。

**一句话：注意力权重决定“取谁的、取多少”，value 向量决定“给什么”。**

同理，其它几个 token 也都用自己的 query 向量得到一行注意力权重，组合出自己的自注意力输出 $\boldsymbol{o}\_i$，$i=2,3,4$；4 个输出按行叠起来，组成一个 $4 \times d\_v$ 的矩阵，即**整句话的新表示**。

原来的 4 个 token 的 embedding 是独立的，不包含上下文信息。经过自注意力层的处理，得到的**新表示**还是 4 个向量（$d\_v$ 维），但不同的是，每个向量包含了上下文信息，并考虑了原 token 的相关性。

## 自注意力计算的矩阵形式 (1.5)

前面几节从 $\boldsymbol{e}\_1$ 的角度描述了如何计算一个 token 的自注意力输出。其它 token 也要进行同样的计算。整个过程可以通过**矩阵运算**的方式**批量**完成。

如前约定：

- 每个 token 的 embedding 是 $d\_{model}$ 维向量；
- $n$ 个 token 按行排成一个 $n \times d\_{model}$ 的矩阵，记作 $\boldsymbol{X}$，$\boldsymbol{X} \in \mathbb{R}^{n \times d\_{model}}$；
- 权重矩阵 $\boldsymbol{W}^Q \in \mathbb{R}^{d\_{model} \times d\_k}$；
- 权重矩阵 $\boldsymbol{W}^K \in \mathbb{R}^{d\_{model} \times d\_k}$；
- 权重矩阵 $\boldsymbol{W}^V \in \mathbb{R}^{d\_{model} \times d\_v}$；

计算各个 token 的 query 向量：

$$
\boldsymbol{Q} = \boldsymbol{X} \boldsymbol{W}^Q
$$

这是第 1.1 节中公式(2)的矩阵形式，一次性计算出所有 token 的 query 向量；矩阵 $\boldsymbol{Q} \in \mathbb{R}^{n \times d\_k}$，每行是一个 token 的 query 向量（$d\_k$ 维）；展开来看：

$$
\boldsymbol{Q} =
\begin{bmatrix}
q\_{11} & q\_{12} & \cdots & q\_{1d\_k} \\\\
q\_{21} & q\_{22} & \cdots & q\_{2d\_k} \\\\
q\_{31} & q\_{32} & \cdots & q\_{3d\_k} \\\\
\vdots  & \vdots  & \ddots & \vdots     \\\\
q\_{n1} & q\_{n2} & \cdots & q\_{nd\_k}
\end{bmatrix} =
\begin{bmatrix}
\boldsymbol{q}\_1 \\\\
\boldsymbol{q}\_2 \\\\
\boldsymbol{q}\_3 \\\\
\vdots            \\\\
\boldsymbol{q}\_n
\end{bmatrix}
$$

同理，计算各个 token 的 key 向量：

$$
\boldsymbol{K} = \boldsymbol{X} \boldsymbol{W}^K
$$

这是第 1.1 节中公式(1)的矩阵形式，一次性计算出所有 token 的 key 向量；矩阵 $\boldsymbol{K} \in \mathbb{R}^{n \times d\_k}$，每行是一个 token 的 key 向量（$d\_k$ 维）。展开：

$$
\boldsymbol{K} =
\begin{bmatrix}
k\_{11} & k\_{12} & \cdots & k\_{1d\_k} \\\\
k\_{21} & k\_{22} & \cdots & k\_{2d\_k} \\\\
k\_{31} & k\_{32} & \cdots & k\_{3d\_k} \\\\
\vdots  & \vdots  & \ddots & \vdots     \\\\
k\_{n1} & k\_{n2} & \cdots & k\_{nd\_k}
\end{bmatrix} =
\begin{bmatrix}
\boldsymbol{k}\_1 \\\\
\boldsymbol{k}\_2 \\\\
\boldsymbol{k}\_3 \\\\
\vdots            \\\\
\boldsymbol{k}\_n
\end{bmatrix}
$$

> 说明：key 向量分量 $k\_{ij}$ 的 $k$ 和 $d\_k$ 的 $k$ 无关。

同理，一次性计算出所有 token 的 value 向量（这是第 1.4 节公式(4)的矩阵形式）：

$$
\boldsymbol{V} = \boldsymbol{X} \boldsymbol{W}^V
$$

矩阵 $\boldsymbol{V} \in \mathbb{R}^{n \times d\_v}$，每行是一个 token 的 value 向量（$d\_v$ 维）。展开来看：

$$
\boldsymbol{V} =
\begin{bmatrix}
v\_{11} & v\_{12} & \cdots & v\_{1d\_v} \\\\
v\_{21} & v\_{22} & \cdots & v\_{2d\_v} \\\\
v\_{31} & v\_{32} & \cdots & v\_{3d\_v} \\\\
\vdots  & \vdots  & \ddots & \vdots     \\\\
v\_{n1} & v\_{n2} & \cdots & v\_{nd\_v}
\end{bmatrix} =
\begin{bmatrix}
\boldsymbol{v}\_1 \\\\
\boldsymbol{v}\_2 \\\\
\boldsymbol{v}\_3 \\\\
\vdots            \\\\
\boldsymbol{v}\_n
\end{bmatrix}
$$

> 说明：value 向量分量 $v\_{ij}$ 的 $v$ 和 $d\_v$ 的 $v$ 无关。

计算所有 token 的注意力得分（把缩放点积，即除以 $\sqrt{d\_k}$，也一起做了）：

$$
\boldsymbol{A} = \frac{\boldsymbol{Q} \boldsymbol{K}^\top}{\sqrt{d\_k}} =
\frac{1}{\sqrt{d\_k}}
\begin{bmatrix}
q\_{11} & q\_{12} & \cdots & q\_{1d\_k} \\\\
q\_{21} & q\_{22} & \cdots & q\_{2d\_k} \\\\
q\_{31} & q\_{32} & \cdots & q\_{3d\_k} \\\\
\vdots  & \vdots  & \ddots & \vdots     \\\\
q\_{n1} & q\_{n2} & \cdots & q\_{nd\_k}
\end{bmatrix}
\begin{bmatrix}
k\_{11} & k\_{21} & k\_{31} & \cdots & k\_{n1} \\\\
k\_{12} & k\_{22} & k\_{32} & \cdots & k\_{n2} \\\\
\vdots  & \vdots  & \vdots  & \ddots & \vdots  \\\\
k\_{1d\_k} & k\_{2d\_k} & k\_{3d\_k} & \cdots & k\_{nd\_k}
\end{bmatrix}
$$

$$
= \begin{bmatrix}
\frac{\boldsymbol{q}\_1 \cdot \boldsymbol{k}\_1}{\sqrt{d\_k}} & \frac{\boldsymbol{q}\_1 \cdot \boldsymbol{k}\_2}{\sqrt{d\_k}} & \cdots & \frac{\boldsymbol{q}\_1 \cdot \boldsymbol{k}\_n}{\sqrt{d\_k}} \\\\
\frac{\boldsymbol{q}\_2 \cdot \boldsymbol{k}\_1}{\sqrt{d\_k}} & \frac{\boldsymbol{q}\_2 \cdot \boldsymbol{k}\_2}{\sqrt{d\_k}} & \cdots & \frac{\boldsymbol{q}\_2 \cdot \boldsymbol{k}\_n}{\sqrt{d\_k}} \\\\
\frac{\boldsymbol{q}\_3 \cdot \boldsymbol{k}\_1}{\sqrt{d\_k}} & \frac{\boldsymbol{q}\_3 \cdot \boldsymbol{k}\_2}{\sqrt{d\_k}} & \cdots & \frac{\boldsymbol{q}\_3 \cdot \boldsymbol{k}\_n}{\sqrt{d\_k}} \\\\
\vdots &  \vdots & \ddots & \vdots         \\\\
\frac{\boldsymbol{q}\_n \cdot \boldsymbol{k}\_1}{\sqrt{d\_k}} & \frac{\boldsymbol{q}\_n \cdot \boldsymbol{k}\_2}{\sqrt{d\_k}} & \cdots & \frac{\boldsymbol{q}\_n \cdot \boldsymbol{k}\_n}{\sqrt{d\_k}}
\end{bmatrix}
$$

这就是第 1.3 节公式(3)的矩阵形式。每个 token 的注意力得分是一个 $n$ 维向量（与 $n$ 个 token 的相关性），所以，结果是一个 $n \times n$ 的矩阵，即 $\boldsymbol{A} \in \mathbb{R}^{n \times n}$，每行代表一个 token 的注意力得分。

矩阵 $\boldsymbol{A}$ 里的元素还是原始得分：第 $i$ 行就是 token $\boldsymbol{e}\_i$ 对所有 token 的注意力得分（缩放后），按第 1.3 节的记法记作 $\alpha\_{ij}$，它们可能不为正，且和不为1；根据第 1.3 节的知识，想要得到**注意力权重**，就得进行归一化，常用的方法是 softmax。显然，每个 token 的注意力得分都要归一化成权重，也就是**要对矩阵** $\boldsymbol{A}$ **的每一行独立进行归一化，行与行之间互不影响**：

$$
w\_{ij} = \frac{\exp(\alpha\_{ij})}{\sum\_{t=1}^n \exp(\alpha\_{it})} \quad \quad i,j=1,2,\dots,n
$$

归一化之后，每行的和为 1；第 $i$ 行就是 token $\boldsymbol{e}\_i$ 分配给全句 $n$ 个 token 的**注意力权重**——第 1 行正是第 1.3 节的 $w\_{1j}$。归一化之后的矩阵记为：

$$
\operatorname{softmax}\left(\boldsymbol{A}\right) = \operatorname{softmax}\left(\frac{\boldsymbol{Q} \boldsymbol{K}^\top}{\sqrt{d\_k}}\right) =
\begin{bmatrix}
w\_{11} & w\_{12} & \cdots & w\_{1n} \\\\
w\_{21} & w\_{22} & \cdots & w\_{2n} \\\\
w\_{31} & w\_{32} & \cdots & w\_{3n} \\\\
\vdots  & \vdots  & \ddots & \vdots  \\\\
w\_{n1} & w\_{n2} & \cdots & w\_{nn}
\end{bmatrix}
$$

$$
\text{对所有 } i=1,2,\dots,n \text{ 满足：} \sum\_{t=1}^n w\_{it} = 1
$$

**注：softmax 作用在矩阵上时，约定为按行归一化，行与行之间互相独立**。

最后，用注意力权重组合 value 向量：

$$
\boldsymbol{O} = \operatorname{softmax}\left(\boldsymbol{A}\right) \boldsymbol{V} =
\operatorname{softmax}\left(\frac{\boldsymbol{Q} \boldsymbol{K}^\top}{\sqrt{d\_k}}\right) \boldsymbol{V}
\tag{6}
$$

公式(6)就是 Transformer 论文（Attention Is All You Need）中的公式(1)。

矩阵 $\boldsymbol{O}$ 维度链是 $(n \times n)(n \times d\_v) = n \times d\_v$，即 $\boldsymbol{O} \in \mathbb{R}^{n \times d\_v}$。把它展开：

$$
\boldsymbol{O} =
\begin{bmatrix}
\sum\_{j=1}^n w\_{1j} v\_{j1} & \sum\_{j=1}^n w\_{1j} v\_{j2} & \sum\_{j=1}^n w\_{1j} v\_{j3} & \cdots & \sum\_{j=1}^n w\_{1j} v\_{jd\_v} \\\\
\sum\_{j=1}^n w\_{2j} v\_{j1} & \sum\_{j=1}^n w\_{2j} v\_{j2} & \sum\_{j=1}^n w\_{2j} v\_{j3} & \cdots & \sum\_{j=1}^n w\_{2j} v\_{jd\_v} \\\\
\sum\_{j=1}^n w\_{3j} v\_{j1} & \sum\_{j=1}^n w\_{3j} v\_{j2} & \sum\_{j=1}^n w\_{3j} v\_{j3} & \cdots & \sum\_{j=1}^n w\_{3j} v\_{jd\_v} \\\\
\vdots                        & \vdots                        & \vdots                        & \ddots & \vdots                           \\\\
\sum\_{j=1}^n w\_{nj} v\_{j1} & \sum\_{j=1}^n w\_{nj} v\_{j2} & \sum\_{j=1}^n w\_{nj} v\_{j3} & \cdots & \sum\_{j=1}^n w\_{nj} v\_{jd\_v}
\end{bmatrix}
$$

和其它矩阵一样，一行对应一个 token。在这里，一行就是一个 token 的自注意力输出。以第一行（对应 token $\boldsymbol{e}\_1$）为例：

- 第 1 行第 1 个元素，即 $\boldsymbol{O}\_{11}$：所有 value 向量的第 1 个分量的线性组合（以 $\boldsymbol{e}\_1$ 的权重组合）；
- 第 1 行第 2 个元素，即 $\boldsymbol{O}\_{12}$：所有 value 向量的第 2 个分量的线性组合（以 $\boldsymbol{e}\_1$ 的权重组合）；
- ......
- 第 1 行第 $d\_v$ 个元素，即 $\boldsymbol{O}\_{1d\_v}$：所有 value 向量的第 $d\_v$ 个分量的线性组合（以 $\boldsymbol{e}\_1$ 的权重组合）；

也就是说，第 1 行的每个分量都是所有 value 向量对应分量的线性组合（以 $\boldsymbol{e}\_1$ 的权重组合），等价于说，**第 1 行是所有 value 向量的线性组合**（以 $\boldsymbol{e}\_1$ 的权重组合），即：

$$
\boldsymbol{o}\_1 = \sum\_{j=1}^n w\_{1j} \boldsymbol{v}\_j
\tag{7}
$$

公式(7)不就是第 1.4 节的公式(5)吗？其它行也一样，只不过组合权重是对应 token 的注意力权重。所以，**公式(6)就是公式(5)的矩阵形式**；可以这样理解：

$$
\boldsymbol{O} =
\begin{bmatrix}
\sum\_{j=1}^n w\_{1j} \boldsymbol{v}\_j \\\\
\sum\_{j=1}^n w\_{2j} \boldsymbol{v}\_j \\\\
\vdots \\\\
\sum\_{j=1}^n w\_{nj} \boldsymbol{v}\_j
\end{bmatrix} =
\begin{bmatrix}
\text{使用} \boldsymbol{e}\_1 \text{的注意力权重线性组合所有 value 向量} \\\\
\text{使用} \boldsymbol{e}\_2 \text{的注意力权重线性组合所有 value 向量} \\\\
\vdots \\\\
\text{使用} \boldsymbol{e}\_n \text{的注意力权重线性组合所有 value 向量}
\end{bmatrix}
$$

> 数学中，常见的是列向量的线性组合按此写法，其实行向量也一样，对应分量线性组合而已。

每一行是对应 token 的注意力权重对所有 value 向量的线性组合。可以表示成下图：

![figure3](self-attention-output.png)
<div style="text-align: center;"><em>Value 向量的线性组合</em></div>

总结起来，Self Attention 的计算就是几个简单的矩阵相乘和一次按行 softmax：

$$
\begin{aligned}
\boldsymbol{Q} & = \boldsymbol{X} \boldsymbol{W}^Q \\\\
\\\\
\boldsymbol{K} & = \boldsymbol{X} \boldsymbol{W}^K \\\\
\\\\
\boldsymbol{V} & = \boldsymbol{X} \boldsymbol{W}^V \\\\
\\\\
\boldsymbol{O} & = \operatorname{softmax}\left(\frac{\boldsymbol{Q} \boldsymbol{K}^\top}{\sqrt{d\_k}}\right) \boldsymbol{V}
\end{aligned}
$$

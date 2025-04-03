---
title: Backtracking算法
date: 2020-04-07 10:28:42
tags: [algorithm, backtracking]
categories: algorithm
---

总结Backtracking算法。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>



# 算法描述 (1)

$$
\textbf{Algorithm: Backtracking}
$$

**输入**: 
    深度$n$：搜索树的最大深度；
    解空间$S$：搜索树每层的选项$choice[d] \in S$；$d \in [0, 1, 2, \ldots, n-1]$

**输出**: 所有满足条件的解

**过程**:

$\text{01:}\hspace{10pt}choice\text{[0:n]} \leftarrow [0, 0, \ldots, 0]\hspace{5pt}\text{//所有层都选择第0个选项}$
$\text{02:}\hspace{10pt}d \leftarrow 0$
$\text{03:}$
$\text{04:}\hspace{10pt}\textbf{while} \hspace{5pt} true \hspace{5pt} \text{\\{}$
$\text{05:}\hspace{20pt}\textbf{if} \hspace{5pt} choice[d] \notin S \hspace{5pt} \text{\\{} \hspace{5pt}\text{//超出S，故d层已无可选项，回溯}$
$\text{06:}\hspace{30pt}choice[d] \leftarrow 0 \hspace{5pt} \text{//回上一层前，重置d层}$
$\text{07:}\hspace{30pt}d \leftarrow d-1 \hspace{5pt} \text{//回上一层}$
$\text{08:}\hspace{30pt}\textbf{if} \hspace{5pt} d \lt 0 \hspace{5pt} \text{\\{} \hspace{5pt} \text{//无可回溯}$
$\text{09:}\hspace{40pt}\textbf{break} \hspace{5pt} \text{//算法结束！}$
$\text{10:}\hspace{30pt}\text{\\}}$
$\text{11:}\hspace{30pt}choice[d] \leftarrow choice[d]+1\hspace{5pt}\text{//选择下一个选项}$
$\text{12:}\hspace{30pt}\textbf{continue}\hspace{5pt}\text{//检查是否超出S，若是则继续回溯}$
$\text{13:}\hspace{20pt}\text{\\}}$
$\text{14:}$
$\text{15:}\hspace{20pt}\textbf{if} \hspace{5pt} choice[0:d]子解合法 \hspace{5pt} \text{\\{}$
$\text{16:}\hspace{30pt}\textbf{if} \hspace{5pt} d=n-1 \hspace{5pt} \text{\\{} \hspace{5pt}\text{//子解合法且到达最底层，解！}$
$\text{17:}\hspace{40pt}\text{emit solution}\hspace{5pt}choice[0:n-1]$
$\text{18:}\hspace{40pt}choice[d] \leftarrow choice[d]+1\hspace{5pt}\text{//继续搜索最底层}$
$\text{19:}\hspace{30pt}\text{\\}} \hspace{5pt} \textbf{else} \hspace{5pt} \text{\\{}$
$\text{20:}\hspace{40pt}d \leftarrow d+1\hspace{5pt}\text{//子解合法但未到最底层，向下}$
$\text{21:}\hspace{30pt}\text{\\}}$
$\text{22:}\hspace{20pt}\text{\\}} \hspace{5pt} \textbf{else} \hspace{5pt} \text{\\{}$
$\text{23:}\hspace{30pt}choice[d] \leftarrow choice[d]+1\hspace{5pt}\text{//子解非法，继续搜索d层}$
$\text{24:}\hspace{20pt}\text{\\}}$
$\text{25:}\hspace{10pt}\text{\\}}$


# 过程图示 (2)

![figure1](backtracking.png)
<div style="text-align: center;"><em>图1</em></div>



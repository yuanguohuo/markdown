---
title: Backtracking算法
date: 2020-04-07 10:28:42
tags: [algorithm, backtracking]
categories: algorithm
---

总结Backtracking算法。

<!-- more -->

# 算法描述 (1)

$$
\textbf{Algorithm: Backtracking}
$$

**输入**: 
    深度$n$：搜索树的最大深度；
    解空间$S$：搜索树每层的选项$choice[d] \in S$；$d \in [0, 1, 2, \ldots, n-1]$

**输出**: 所有满足条件的解

**过程**:

$\text{01:}\hspace{10pt}choice\text{[0:n-1]} \leftarrow [0, 0, \ldots, 0]\hspace{5pt}\text{//所有层都选择第0个选项}$<br>
$\text{02:}\hspace{10pt}d \leftarrow 0$<br>
$\text{03:}$<br>
$\text{04:}\hspace{10pt}\textbf{while} \hspace{5pt} d \geq 0 \hspace{5pt} \text{\\{}$<br>
$\text{05:}\hspace{20pt}\textbf{if} \hspace{5pt} choice[d] \notin S \hspace{5pt} \text{\\{} \hspace{5pt}\text{//超出S，故d层已无可选项，回溯}$<br>
$\text{06:}\hspace{30pt}choice[d] \leftarrow 0 \hspace{5pt} \text{//回上一层前，重置d层}$<br>
$\text{07:}\hspace{30pt}d \leftarrow d-1 \hspace{5pt} \text{//回上一层}$<br>
$\text{08:}\hspace{30pt}\textbf{if} \hspace{5pt} d \geq 0 \hspace{5pt} \text{\\{} \hspace{5pt} \text{//还有上一层}$<br>
$\text{09:}\hspace{40pt}choice[d] \leftarrow choice[d]+1\hspace{5pt}\text{//选择下一个选项}$<br>
$\text{10:}\hspace{30pt}\text{\\}}$<br>
$\text{11:}\hspace{30pt}\textbf{continue}\hspace{5pt}\text{//若有上一层，已回溯且选择了新选项；若无上一层，则算法结束}$<br>
$\text{12:}\hspace{20pt}\text{\\}}$<br>
$\text{13:}$<br>
$\text{14:}\hspace{20pt}\textbf{if} \hspace{5pt} choice[0:d]子解合法 \hspace{5pt} \text{\\{}$<br>
$\text{15:}\hspace{30pt}\textbf{if} \hspace{5pt} d=n-1 \hspace{5pt} \text{\\{} \hspace{5pt}\text{//子解合法且到达最底层，解！}$<br>
$\text{16:}\hspace{40pt}\text{emit solution}\hspace{5pt}choice[0:n-1]$<br>
$\text{17:}\hspace{40pt}choice[d] \leftarrow choice[d]+1\hspace{5pt}\text{//继续搜索最底层}$<br>
$\text{18:}\hspace{30pt}\text{\\}} \hspace{5pt} \textbf{else} \hspace{5pt} \text{\\{}$<br>
$\text{19:}\hspace{40pt}d \leftarrow d+1\hspace{5pt}\text{//子解合法但未到最底层，向下}$<br>
$\text{20:}\hspace{30pt}\text{\\}}$<br>
$\text{21:}\hspace{20pt}\text{\\}} \hspace{5pt} \textbf{else} \hspace{5pt} \text{\\{}$<br>
$\text{22:}\hspace{30pt}choice[d] \leftarrow choice[d]+1\hspace{5pt}\text{//子解非法，继续搜索d层}$<br>
$\text{23:}\hspace{20pt}\text{\\}}$<br>
$\text{24:}\hspace{10pt}\text{\\}}$


# 过程图示 (2)

![figure1](backtracking.png)
<div style="text-align: center;"><em>图1</em></div>



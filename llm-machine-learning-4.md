---
title: 机器学习原理-分类(判别式)
date: 2025-11-25 18:20:55
tags: [llm,machine-learning]
categories: llm
---

机器学习主要分为监督学习（Supervised Learning）、无监督学习（Unsupervised Learning）、强化学习（Reinforcement Learning）以及半监督学习（Semi-supervised Learning）等范式。

监督学习主要包括两大类基础任务：回归（Regression）和分类（Classification）。回归用于预测连续值（输出为实数），分类则用于预测离散类别标签（输出为有限集合中的某个类别）。此外，结构化学习（Structured Learning）是一类更复杂的监督学习任务，其输出是具有内部结构的对象（如序列、树等），可视为分类或回归的高维扩展。

在分类任务中，模型可分为生成式模型（Generative Models）和判别式模型（Discriminative Models）。生成式模型通过对联合分布 P(x,y) 建模来实现分类，典型方法包括高斯判别分析（GDA）及其两种常见形式——假设协方差矩阵共享的线性判别分析（LDA）和允许协方差矩阵独立的二次判别分析（QDA），以及朴素贝叶斯等。判别式模型则直接建模条件分布 P(y∣x)，代表方法有逻辑回归（Logistic Regression）、支持向量机（SVM）等。

本文将简要介绍逻辑回归 Logistic Regression。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

# 问题设定 (1)

训练数据为一组带标签的样本。

- **对象**：由 $d$ 个连续特征描述，表示为特征向量 $\boldsymbol{x} \in \mathbb{R}^d$；
- **标签**：样本所属的类别，记为 $y$，其中 $y \in \{c_1, c_2, \dots, c_K\}$（离散变量，共 $K$ 个类别）后文使用$k$表示一个具体的类别，$y$表示类别变量；
- **样本**：一个对象及其所属的类别构成一个样本，记为 $(\boldsymbol{x}_i, y_i)$；
- **样本集**：给定 $N$ 个独立同分布的训练样本

  $$
  (\boldsymbol{x}_1, y_1),\ (\boldsymbol{x}_2, y_2),\ \dots,\ (\boldsymbol{x}_N, y_N)
  $$
  其中 $\boldsymbol{x}_i \in \mathbb{R}^d$ 是第 $i$ 个样本的特征向量，$y_i$ 是其对应的类别标签。

> **注**：下标 $i$ 表示**样本索引**，而非向量分量；向量 $\boldsymbol{x}\_i$ 的第 $j$ 个分量记为 $x\_{ij}$。

**任务**：学习一个分类模型，使其能够对新的输入 $\boldsymbol{x}_{\text{new}}$ 预测其所属类别 $y$。

# 核心思想 (2)

在[上一章](https://www.yuanguohuo.com/2025/11/20/llm-machine-learning-3/)中，LDA（Linear Discriminant Analysis）通过训练样本，估计每个类别 $k$ 的先验概率 $\pi_k$、均值向量 $\boldsymbol{\mu}_k$ 以及所有类别共享的协方差矩阵 $\boldsymbol{\Sigma}$，然后根据贝叶斯公式计算新特征向量的后验概率。

据其第7节可知，后验概率可表示为：

$$
P(y \mid \boldsymbol{x}) = \operatorname{softmax}(\boldsymbol{z}),
\quad \text{其中} \quad
z_k = \boldsymbol{w}_k^\top \boldsymbol{x} + b_k,
$$

而 $\boldsymbol{w}_k$ 和 $b_k$ 由 $\pi_k$、$\boldsymbol{\mu}_k$ 和 $\boldsymbol{\Sigma}$ 解析确定。

既然如此，何不跳过对数据生成过程的假设（如特征服从高斯分布），**直接假设后验概率具有这种 softmax 形式**，并将 $\boldsymbol{w}_k$、$b_k$ 视为待学习的参数？

这正是逻辑回归（Logistic Regression）的核心思想：它是一种判别式模型，直接对条件概率 $P(y \mid \boldsymbol{x})$ 建模，并通过最大似然估计（等价于最小化交叉熵损失）从数据中学习参数。





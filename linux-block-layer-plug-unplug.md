---
title: block layer的plug和unplug
date: 2019-10-04 23:29:09
tags: [kernel,io,block]
categories: linux 
---

本篇研究一个具体问题：block device stack中，如何避免递归。

<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

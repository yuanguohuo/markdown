---
title: Wandering Tree
date: 2019-07-06 13:12:20
tags: [tree]
categories: algorithm
---


<!-- more -->

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>





There is a whitepaper on the UBIFS page, and LWN wrote about it in the LogFS article.
The on-flash tree looks much like the structure used by ext2. There are some differences in how it is managed, however. The log structure of the filesystem implies that blocks cannot be rewritten in place; any time a block is changed it must be moved and written to a new location. If there are pointers to the moved block (think about the usual indirect blocks used to store the layout of larger files), the blocks containing the pointers must also be changed, and thus moved. That, in turn, will require changes at the next level up in the tree. Thus changes at the bottom of the tree will propagate upward all the way to the root. This is the "wandering tree" algorithm. One of the advantages is that the old filesystem structure remains valid until the root is rewritten - a crash could cause the loss of the last operation, but it will leave previous data and the structure of the filesystem intact.
Actually managing the entire directory tree as a wandering tree would be expensive; beyond that, files with multiple hard links break the tree structure and make wandering trees much harder to implement. So the actual tree implemented by LogFS just has two levels. There is an "inode file" containing the inode structures for every file and directory existing within the filesystem; each inode then points to the associated blocks holding the file's data. Directory entries contain a simple integer index giving the inode offset within the inode file. So changes to an inode only require writing the inode itself and the inode file; the rest of the directory structure need not be touched.

---
title: Linux Ftrace 
date: 2023-07-18 19:13:21
tags: [kernel, tracing, tracer]
categories: tracing
---

Linux tracing系统中有3层：front-end, tracing-framework(本文叫tracer)和event-source; 本文聚焦于ftrace，它属于tracer，类似的还有perf, eBPF等。

<!-- more -->

# 框架 (1)

先引用[两张图](https://leezhenghui.github.io/linux/2019/03/05/exploring-usdt-on-linux.html)说明一下ftrace在linux tracing中的位置与角色：

![figure1](linux-tracing-tracing-overview.png)
<div style="text-align: center;"><em>图1</em></div>

Tracing系统包含3层: event-source是数据来源，包括tracepoint, kprobe, usdt等；tracer (tracing framework) 运行于内核中，负责搜集event数据，甚至能够聚合与统计数据(例如eBPF)；front-end为用户提供交互接口，触发tracing，获取数据并进行聚合与统计(tracer不能做的话)。具体到ftrace，是这样的：

![figure2](linux-tracing-ftrace.png)
<div style="text-align: center;"><em>图2</em></div>

Front-end包括trace-cmd和tracefs `/sys/kernel/debug/tracing/`；tracing-framework层包含event-tracer, function-tracer和latency-tracer等；event-source也有多种，例如tracepoint, kprobe, uprobe是event-tracer的数据源，`mcount()`是function-tracer的数据源。

# set up ftrace (2)

Ftrace的接口在debugfs中，通常mount在`/sys/kernel/debug`；若配置了ftrace，它就会重建一个`/sys/kernel/debug/tracing`目录。以debug内核为目的，需要配置这些选项：

- CONFIG_FUNCTION_TRACER
- CONFIG_FUNCTION_GRAPH_TRACER
- CONFIG_STACK_TRACER
- CONFIG_DYNAMIC_FTRACE

# function-tracer (3)

如图"ftrace-component-stack"中所示，function-tracer是ftrace的一种tracer，也是ftrace的最强大的tracer；它的工作原理是：使用gcc的`-pg`选项，编译时，让每个内核函数都调用`mcount()`函数(使用汇编实现，不遵守C ABI)。编译时`mcount()`的调用位置被记录到一个列表中。当配置了`CONFIG_DYNAMIC_FTRACE`，boot时就把列表中所有对`mcount()`的调用转化为`NOP`指令，所以系统不损失任何性能。当启用function-tracer时，再使用列表，把`NOP`指令替换成对trace的调用。所以，推荐配置`CONFIG_DYNAMIC_FTRACE`，因为这样才能使系统有100%的性能。

# event-tracer (4)

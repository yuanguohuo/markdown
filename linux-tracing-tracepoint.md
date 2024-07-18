---
title: Linux Tracepoint
date: 2024-07-18 18:30:32
tags: [kernel, tracing, event-source]
categories: tracing
---

Linux tracing中有3层：front-end, tracer(即tracing framework)和event-source. Tracepoint属于event-source，是一种kernel static tracing事件源。本文介绍tracepoint的实现以及使用方式，特别是如何被ftrace使用。

<!-- more -->

# 概述 (1)

广义的tracepoint指静态tracepoint(本文内容)和动态tracepoint(包括kprobe和uprobe)。狭义的tracepoint就是静态tracepoint；它在内核代码中就叫tracepoint，实例由结构体`struct tracepoint __tracepoint_##name`表示(其中`##name`是`##subsys_##eventname`)；hook函数是`trace_##name()`，即`trace_##subsys_##eventname()`。静态的意思是，预先在内核代码中调用hook函数。所以它只能trace这些编译到内核中的hook点。下文的tracepoint都是指狭义的静态tracepoint.

内核的基础框架部分包含一套tracepoint机制，用于何创建与使用tracepoint实例。Subsystem及kernel-module开发者使用这套机制创建tracepoint实例`__tracepoint_##subsys_##eventname`并在重要的内核代码路径上调用`trace_##subsys_##eventname()`函数，即放置hook；hook函数用于调用probe函数(也叫callback函数，在本文中这两个词是等价的)；probe函数是使用者在运行时提供的。也就是说，hook是静态的，probe是动态的。Tracepoint实例可以是`on`状态(hook关联着一个或多个probe)，也可以是`off`状态(hook不与任何probe关联)。处于`off`状态的tracepoint不起作用，除了增加一点时间开销(checking a condition for a branch)和一点空间开销(adding a few bytes for the function call at the end of the instrumented function and adds a data structure in a separate section)；而当tracepoint在`on`状态时，每次执行到hook时，probe函数就被调用一次。

显然，创建tracepoint与放置hook是内核开发者(subsys及module开发者)的事，使用tracepoint(实现probe并与hook关联)是系统维护者，或者任何要学习、定位系统bug、追踪系统性能的人员的事。

# 创建tracepoint (2)

严格来说，tracepoint是由`DECLARE_TRACE/DEFINE_TRACE`创建的。但目前内核已不直接使用它来创建tracepoint，而是通过下文介绍的`TRACE_EVENT`宏。然而`DECLARE_TRACE/DEFINE_TRACE`是`TRACE_EVENT`的基础：`TRACE_EVENT`展开后包含`DECLARE_TRACE/DEFINE_TRACE`。`DECLARE_TRACE/DEFINE_TRACE`能够完整的创建tracepoint；`TRACE_EVENT`是在创建的tracepoint之后，为使用它提供一些便利。

创建tracepoint分2步：declare和define，分别有`DECLARE_TRACE`和`DEFINE_TRACE`完成。内核开发者这样使用它们(例如开发foo subsystem，创建bar tracepoint)：

- declare tracepoint

在header文件`include/trace/events/foo.h`中使用`DECLARE_TRACE`声明tracepoint:

```C
#undef TRACE_SYSTEM
#define TRACE_SYSTEM subsys

#if !defined(_TRACE_SUBSYS_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_SUBSYS_H

#include <linux/tracepoint.h>

DECLARE_TRACE(foo_bar,
        TP_PROTO(int firstarg, struct task_struct *p),
        TP_ARGS(firstarg, p));

#endif /* _TRACE_SUBSYS_H */

/* This part must be outside protection */
#include <trace/define_trace.h>
```

宏`DECLARE_TRACE`定义在`include/linux/tracepoint.h`中，以linux-5.10.161为例:

```C
#define DECLARE_TRACE(name, proto, args)                                \
        __DECLARE_TRACE(name, PARAMS(proto), PARAMS(args),              \
                        cpu_online(raw_smp_processor_id()),             \
                        PARAMS(void *__data, proto),                    \
                        PARAMS(__data, args))


#define __DECLARE_TRACE(name, proto, args, cond, data_proto, data_args) \
        extern int __traceiter_##name(data_proto);                      \
        DECLARE_STATIC_CALL(tp_func_##name, __traceiter_##name);        \
        extern struct tracepoint __tracepoint_##name;                   \
        static inline void trace_##name(proto)                          \
        {                                                               \
                if (static_key_false(&__tracepoint_##name.key))         \
                        __DO_TRACE(name,                                \
                                TP_PROTO(data_proto),                   \
                                TP_ARGS(data_args),                     \
                                TP_CONDITION(cond), 0);                 \
                if (IS_ENABLED(CONFIG_LOCKDEP) && (cond)) {             \
                        rcu_read_lock_sched_notrace();                  \
                        rcu_dereference_sched(__tracepoint_##name.funcs);\
                        rcu_read_unlock_sched_notrace();                \
                }                                                       \
        }                                                               \
        __DECLARE_TRACE_RCU(name, PARAMS(proto), PARAMS(args),          \
                PARAMS(cond), PARAMS(data_proto), PARAMS(data_args))    \
        static inline int                                               \
        register_trace_##name(void (*probe)(data_proto), void *data)    \
        {                                                               \
                return tracepoint_probe_register(&__tracepoint_##name,  \
                                                (void *)probe, data);   \
        }                                                               \
        static inline int                                               \
        register_trace_prio_##name(void (*probe)(data_proto), void *data,\
                                   int prio)                            \
        {                                                               \
                return tracepoint_probe_register_prio(&__tracepoint_##name, \
                                              (void *)probe, data, prio); \
        }                                                               \
        static inline int                                               \
        unregister_trace_##name(void (*probe)(data_proto), void *data)  \
        {                                                               \
                return tracepoint_probe_unregister(&__tracepoint_##name,\
                                                (void *)probe, data);   \
        }                                                               \
        static inline void                                              \
        check_trace_callback_type_##name(void (*cb)(data_proto))        \
        {                                                               \
        }                                                               \
        static inline bool                                              \
        trace_##name##_enabled(void)                                    \
        {                                                               \
                return static_key_false(&__tracepoint_##name.key);      \
        }
```

可见，这里定义了一些函数：

- `trace_foo_bar()`: 放置hook；
- `register_trace_foo_bar()/unregister_trace_foo_bar()`: 关联/解绑probe;
- `trace_foo_bar_enabled()`: tracepoint是否开启；


除此之外，还声明了一个`extern struct tracepoint`类型的结构体`__tracepoint_foo_bar`(注意带有`extern`，只声明，没有定义). 类型`struct tracepoint`定义在`include/linux/tracepoint-defs.h`(linux-5.10.161):

```C
struct tracepoint {
        const char *name;               /* Tracepoint name */
        struct static_key key;          /*用于确定是否enabled*/
        struct static_call_key *static_call_key;
        void *static_call_tramp;
        void *iterator;
        int (*regfunc)(void);                /*在add probe之前调用*/
        void (*unregfunc)(void);             /*在remove probe之后调用?*/
        struct tracepoint_func __rcu *funcs; /*probe函数, 允许有多个*/
};
```

- define tracepoint


在C文件`foo/xx.c`中使用`DEFINE_TRACE`define tracepoint:

```
#include <trace/events/subsys.h>

#define CREATE_TRACE_POINTS
DEFINE_TRACE(subsys_eventname);
```

声明中带有`extern`，所以没有分配`struct tracepoint __tracepoint_foo_bar`，这里才真正分配这个变量，它被存储在`__tracepoints` section中。以linux-5.10.161为例，`DEFINE_TRACE`展开为(在`include/linux/tracepoint.h`中)：

```C
#define DEFINE_TRACE(name, proto, args)                                 \
        DEFINE_TRACE_FN(name, NULL, NULL, PARAMS(proto), PARAMS(args));

#define DEFINE_TRACE_FN(_name, _reg, _unreg, proto, args)               \
        static const char __tpstrtab_##_name[]                          \
        __section("__tracepoints_strings") = #_name;                    \
        extern struct static_call_key STATIC_CALL_KEY(tp_func_##_name); \
        int __traceiter_##_name(void *__data, proto);                   \
                                                                        \
        /*分配__tracepoint_foo_bar，在__tracepoints段*/                 \
        struct tracepoint __tracepoint_##_name  __used                  \
        __section("__tracepoints") = {                                  \
                .name = __tpstrtab_##_name,                             \
                .key = STATIC_KEY_INIT_FALSE,                           \
                .static_call_key = &STATIC_CALL_KEY(tp_func_##_name),   \
                .static_call_tramp = STATIC_CALL_TRAMP_ADDR(tp_func_##_name), \
                .iterator = &__traceiter_##_name,                       \
                .regfunc = _reg,                                        \
                .unregfunc = _unreg,                                    \
                .funcs = NULL };                                        \
                                                                        \
        /*调用trace_foo_bar，最终会调此函数：逐个调用funcs中的probe*/   \
        __TRACEPOINT_ENTRY(_name);                                      \
        int __traceiter_##_name(void *__data, proto)                    \
        {                                                               \
                struct tracepoint_func *it_func_ptr;                    \
                void *it_func;                                          \
                                                                        \
                it_func_ptr =                                           \
                        rcu_dereference_raw((&__tracepoint_##_name)->funcs); \
                if (it_func_ptr) {                                      \
                        do {                                            \
                                it_func = (it_func_ptr)->func;          \
                                __data = (it_func_ptr)->data;           \
                                ((void(*)(void *, proto))(it_func))(__data, args); \
                        } while ((++it_func_ptr)->func);                \
                }                                                       \
                return 0;                                               \
        }                                                               \
        DEFINE_STATIC_CALL(tp_func_##_name, __traceiter_##_name);
```

- 放置hook

在需要放置tracepoint的地方，例如`foo/yy.c`中：

```
void some_kernel_func(void)
{
        ...
        trace_foo_bar(arg, task);
        ...
}
```

如前所述，最后会逐个调用probe；使用`register_trace_foo_bar()`关联probe，使用`unregister_trace_foo_bar()`解除关联的probe。

这种方式创建了一个叫`bar`的tracepoint，隶属于`foo` subsystem；要使用它，使用者必须提供probe，内核开发者没有提供任何默认probe；要想注册probe，需要开发一个内核模块，在模块中实现probe，并通过`register_trace_foo_bar/unregister_trace_foo_bar`关联或解除关联。所以，不方便使用。我看了几个版本，3.19.8，5.10.161以及6.4.9，都极少直接使用`DECLARE_TRACE/DEFINE_TRACE`：只有`sched.h`使用它创建了几个tracepoint，还是"for testing and debugging purposes"。

# 作为ftrace的event-source (3)

## `TRACE_EVENT`的结构 (3.1)

当前内核版本中，tracepoint自动作为ftrace的一个event-source，所以也叫**trace event**。注意，这不意味着tracepoint和ftrace绑定或者耦合了：ftrace只是可以使用它的tracer之一，其它tracer，例如perf, LTTng, SystemTap都可以使用tracepoint；tracepoint对使用它的tracer是无感知的。

鉴于自动作为ftrace的event-source，当前内核提供了一个新的宏来创建tracepoint，它就是`TRACE_EVENT`，强调它是一个**event**。可以这么理解，为了方便使用，`TRACE_EVENT`在创建tracepoint的同时，还自动为其提供一个probe实现：这个probe可以直接被ftrace使用(ftrace属于tracer)，在第3.3节中将看到这个probe.

`TRACE_EVENT`的目标是：

- Target-A: It must create a tracepoint (指的是hook) that can be placed in the kernel code.
- Target-B: It must create a callback(即probe) function that can be hooked to this tracepoint.
- Target-C: The callback(probe) function must be able to record the data passed to it into the tracer ring buffer in the fastest way possible. 即callback/probe函数必须能够把它的入参记录到tracer ring buffer中去(tracer就是ftrace，所以就是记录到ftrace ring buffer中)。
- Target-D: It must create a function that can parse the data recorded to the ring buffer and translate it to a human readable format that the tracer can display to a user.

为了达到这些目标，`TRACE_EVENT`宏被设计成六部分，对应它的六个参数`TRACE_EVENT(name, proto, args, struct, assign, print)`:

- Param-1 (`name`): the name of the tracepoint to be created.
- Param-2 (`prototype`): the prototype for the tracepoint callback(probe)；
- Param-3 (`args`): the arguments that match the prototype.
- Param-4 (`struct`): the structure that a tracer could use (but is not required to) to store the data passed into the tracepoint；Target-C说，callback/probe函数必须能够把它的入参记录到ftrace ring buffer中；这个结构体就是在ftrace ring buffer中的entry的类型；注意，这里定义的只是一些字段，不是整个结构体，`TRACE_EVENT`会把它展开到一个结构体中(结构体还有其他字段)；一个真实的例子是`struct trace_event_raw_sched_switch`(见第3.3节，第3次展开)；
- Param-5 (`assign`): the C-like way to assign the data to the structure. 实现Target-C：callback/probe在ftrace ring buffer中创建一个entry(类型如上`struct`)，然后调用`assign`给entry赋值。所以`assign`不是probe函数，也应该是probe函数的组成部分(果然：见第3.3节，第8次展开)。
- Param-6 (`print`): the way to output the structure in human readable ASCII format. 实现Target-D: 把entry(类型如上`struct`)转化成可读形式，写入用户态output buffer(见第3.3节，第5次展开)。

## 例子 (3.2)

从本节开始不再以`foo_bar`为例，而是使用一个真实的例子`sched_switch`，这样更有体感。它的定义在`include/trace/events/sched.h`中(linux-5.10.161):

```C
TRACE_EVENT(sched_switch,

        TP_PROTO(bool preempt,
                 struct task_struct *prev,
                 struct task_struct *next),

        TP_ARGS(preempt, prev, next),

        TP_STRUCT__entry(
                __array(        char,   prev_comm,      TASK_COMM_LEN   )
                __field(        pid_t,  prev_pid                        )
                __field(        int,    prev_prio                       )
                __field(        long,   prev_state                      )
                __array(        char,   next_comm,      TASK_COMM_LEN   )
                __field(        pid_t,  next_pid                        )
                __field(        int,    next_prio                       )
        ),

        TP_fast_assign(
                memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
                __entry->prev_pid       = prev->pid;
                __entry->prev_prio      = prev->prio;
                __entry->prev_state     = __trace_sched_switch_state(preempt, prev);
                memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
                __entry->next_pid       = next->pid;
                __entry->next_prio      = next->prio;
                /* XXX SCHED_DEADLINE */
        ),

        TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> next_comm=%s next_pid=%d next_prio=%d",
                __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,

                (__entry->prev_state & (TASK_REPORT_MAX - 1)) ?
                  __print_flags(__entry->prev_state & (TASK_REPORT_MAX - 1), "|",
                                { TASK_INTERRUPTIBLE, "S" },
                                { TASK_UNINTERRUPTIBLE, "D" },
                                { __TASK_STOPPED, "T" },
                                { __TASK_TRACED, "t" },
                                { EXIT_DEAD, "X" },
                                { EXIT_ZOMBIE, "Z" },
                                { TASK_PARKED, "P" },
                                { TASK_DEAD, "I" }) :
                  "R",

                __entry->prev_state & TASK_REPORT_MAX ? "+" : "",
                __entry->next_comm, __entry->next_pid, __entry->next_prio)
);

```

先演练一下：

```bash
$ echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
$ head -n 20 /sys/kernel/debug/tracing/trace

# tracer: nop
#
# entries-in-buffer/entries-written: 18375/18375   #P:16
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
          <idle>-0       [012] d... 197376.159374: sched_switch: prev_comm=swapper/12 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/u32:1 next_pid=21084 next_prio=120
   kworker/u32:1-21084   [012] d... 197376.159382: sched_switch: prev_comm=kworker/u32:1 prev_pid=21084 prev_prio=120 prev_state=I ==> next_comm=swapper/12 next_pid=0 next_prio=120
          <idle>-0       [008] d... 197376.159387: sched_switch: prev_comm=swapper/8 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=sshd next_pid=21056 next_prio=120
            sshd-21056   [008] d... 197376.159409: sched_switch: prev_comm=sshd prev_pid=21056 prev_prio=120 prev_state=S ==> next_comm=swapper/8 next_pid=0 next_prio=120
          <idle>-0       [012] d... 197376.159480: sched_switch: prev_comm=swapper/12 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/u32:1 next_pid=21084 next_prio=120
            bash-21058   [010] d... 197376.159481: sched_switch: prev_comm=bash prev_pid=21058 prev_prio=120 prev_state=S ==> next_comm=swapper/10 next_pid=0 next_prio=120
   kworker/u32:1-21084   [012] d... 197376.159485: sched_switch: prev_comm=kworker/u32:1 prev_pid=21084 prev_prio=120 prev_state=I ==> next_comm=swapper/12 next_pid=0 next_prio=120
          <idle>-0       [008] d... 197376.159491: sched_switch: prev_comm=swapper/8 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=sshd next_pid=21056 next_prio=120
            sshd-21056   [008] d... 197376.159506: sched_switch: prev_comm=sshd prev_pid=21056 prev_prio=120 prev_state=S ==> next_comm=swapper/8 next_pid=0 next_prio=120

```

猜测：

- hook函数调用probe函数，它们的prototype一样；
- probe函数在ftrace ring buffer中构造一个entry，类型是`TP_STRUCT__entry`；
- probe函数调用`assign`给entry赋值；
- ftrace调用`TP_printk`显示trace到的信息；

通过后文可知：

- hook函数还是`__DECLARE_TRACE`中的`trace_##name`展开的，所以hook的prototype是：`trace_sched_switch(bool preempt, struct task_struct *prev, struct task_struct *next);` probe只比hook多了一个参数；
- 为什么还要`TP_ARGS(preempt, prev, next)`呢？因为有了它hook调用probe就方便了：`probe(..., preempt, prev, next);`
- `TP_STRUCT__entry`里定义了一些struct的字段，不是完整的struct；它们将多次被展开到不同的struct里。它们最终展开成啥样，取决于`__array`, `__field`等宏如何实现；在`struct trace_event_raw_sched_switch`中，展开的结果就是字面的样子，这个struct也就是entry的结构。
- 另外，`sched_switch`例子也比较简单，没有`__dynamic_array`类型的字段；dynamic字段size不确定，所以它们在结构体中的offset也需要动态处理(本文不考虑)；

第3.3节将回顾这个猜测，做更清晰的解释。

## `TRACE_EVENT`的实现 (3.3)

宏`TRACE_EVENT`一共被展开9次。

### 第1次展开 (3.3.1)

在`include/linux/tracepoint.h`中(linux-5.10.161):

```C
#define TRACE_EVENT(name, proto, args, struct, assign, print)   \
        DECLARE_TRACE(name, PARAMS(proto), PARAMS(args))
```

### 第2次展开 (3.3.2)

在`include/trace/define_trace.h`中(linux-5.10.161):

```C
#define TRACE_EVENT(name, proto, args, tstruct, assign, print)  \
        DEFINE_TRACE(name, PARAMS(proto), PARAMS(args))
```

等等，这不就是第2节`DECLARE_TRACE/DEFINE_TRACE`的工作吗？没错，`TRACE_EVENT`的首要任务还是创建tracepoint；然后，它才能让tracepoint作为ftrace的event。这就是接下来的工作。

### 高阶宏 (3.3.3)

在介绍`TRACE_EVENT`如何把tracepoint变成ftrace的event之前，先看一个C语言中关于宏的玩法，我把它做高阶宏(HOM: High-Order-Macro)：

```C
#define DOGS { HOM(JACK_RUSSELL), HOM(BULL_TERRIER), HOM(ITALIAN_GREYHOUND) }

#undef HOM
#define HOM(a) ENUM_##a
enum dog_enums DOGS;

#undef HOM
#define HOM(a) #a
char *dog_strings[] = DOGS;

char *dog_to_string(enum dog_enums dog)
{
        return dog_strings[dog];
}
```

首先定义了`DOGS`这个宏，然后展开它2次。由于每次高阶宏`HOM`的定义不同，所以展开的结果也不同，分别为：

```C
enum dog_enums { ENUM_JACK_RUSSELL, ENUM_BULL_TERRIER, ENUM_ITALIAN_GREYHOUND };
```

```C
char *dog_strings[] = { "JACK_RUSSELL", "BULL_TERRIER", "ITALIAN_GREYHOUND" };
```

看清楚了这两点，最后函数`dog_to_string()`的作用显而易见。

`TRACE_EVENT`的剩下的7次定义，就是反复使用这个trick。都在`include/trace/trace_events.h`(linux-5.10.161)中：

首先，把`TRACE_EVENT`定义为高阶宏`DECLARE_EVENT_CLASS`和`DEFINE_EVENT`:

```C
#define TRACE_EVENT(name, proto, args, tstruct, assign, print) \
        DECLARE_EVENT_CLASS(name,                              \
                             PARAMS(proto),                    \
                             PARAMS(args),                     \
                             PARAMS(tstruct),                  \
                             PARAMS(assign),                   \
                             PARAMS(print));                   \
        DEFINE_EVENT(name, name, PARAMS(proto), PARAMS(args));
```

接下来，每次重新定义高阶宏，就定义了新的`TRACE_EVENT`，共7次，都在这个header文件中。接着前2次，从第3次开始记：

### 第3次展开 (3.3.4)

```C
#undef __field
#define __field(type, item)             type    item;

#undef __field_ext
#define __field_ext(type, item, filter_type)    type    item;

#undef __field_struct
#define __field_struct(type, item)      type    item;

#undef __field_struct_ext
#define __field_struct_ext(type, item, filter_type)     type    item;

#undef __array
#define __array(type, item, len)        type    item[len];

#undef __dynamic_array
#define __dynamic_array(type, item, len) u32 __data_loc_##item;

#undef __string
#define __string(item, src) __dynamic_array(char, item, -1)

#undef __bitmask
#define __bitmask(item, nr_bits) __dynamic_array(char, item, -1)

#undef TP_STRUCT__entry
#define TP_STRUCT__entry(args...) args

#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(name, proto, args, tstruct, assign, print)  \
        struct trace_event_raw_##name {                                 \
                struct trace_entry      ent;                            \
                tstruct                                                 \
                char                    __data[0];                      \
        };                                                              \
                                                                        \
        static struct trace_event_class event_class_##name;

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, name, proto, args)       \
        static struct trace_event_call  __used          \
        __attribute__((__aligned__(4))) event_##name
```

以`sched_switch`为例，展开的结果是:

```C
struct trace_event_raw_sched_switch {
  struct trace_entry    ent;

  /*展开自TP_STRUCT__entry参数，见include/trace/events/sched.h中sched_switch的定义*/
  char   prev_comm[TASK_COMM_LEN];
  pid_t  prev_pid;
  int    prev_prio;
  long   prev_state;
  char   next_comm[TASK_COMM_LEN];
  pid_t  next_pid;
  int    next_prio;

  char   __data[0];
};

static struct trace_event_class event_class_sched_switch;

static struct trace_event_call  __used  __attribute__((__aligned__(4))) event_sched_switch;
```

第3.1节中Param-4提到的structure就在此定义，即`struct trace_event_raw_sched_switch`; probe函数会把它的入参中有用的信息保存到这个结构体中(通过`assign`函数实现)；由于`sched_switch`比较简单，这个structure中没有dynamic字段。若有的话，structure中会展开有``__data_loc_##item`字段；字段的的真实数据会存在`__data`中，`__data_loc_##item`记录它们的信息。

结构体`struct trace_event_call event_sched_switch`是整个event信息的入口(貌似其他tracing机制，例如kprobe, uprobe, perf的event-source也使用它)，定义在`include/linux/trace_events.h`(linux-5.10.161)中：

```C
struct trace_event_call {
        struct list_head        list;
        struct trace_event_class *class;
        union {
                char                    *name;
                /* Set TRACE_EVENT_FL_TRACEPOINT flag when using "tp" */
                struct tracepoint       *tp;
        };
        struct trace_event      event;
        char                    *print_fmt;
        struct event_filter     *filter;
        void                    *mod;
        void                    *data;
        /*
         *   bit 0:             filter_active
         *   bit 1:             allow trace by non root (cap any)
         *   bit 2:             failed to apply filter
         *   bit 3:             trace internal event (do not enable)
         *   bit 4:             Event was enabled by module
         *   bit 5:             use call filter rather than file filter
         *   bit 6:             Event is a tracepoint
         */
        int                     flags; /* static flags of different events */

#ifdef CONFIG_PERF_EVENTS
        int                             perf_refcount;
        struct hlist_head __percpu      *perf_events;
        struct bpf_prog_array __rcu     *prog_array;

        int     (*perf_perm)(struct trace_event_call *,
                             struct perf_event *);
#endif
};
```

类型`struct trace_event_class`也定义在`include/linux/trace_events.h`(linux-5.10.161)中(后文将看到perf event-source也使用它)：

```C
struct trace_event_class {
        const char              *system;
        void                    *probe;
#ifdef CONFIG_PERF_EVENTS
        void                    *perf_probe;
#endif
        int                     (*reg)(struct trace_event_call *event,
                                       enum trace_reg type, void *data);
        struct trace_event_fields *fields_array;
        struct list_head        *(*get_fields)(struct trace_event_call *);
        struct list_head        fields;
        int                     (*raw_init)(struct trace_event_call *);
};
```

### 第4次展开 (3.3.5)

```C
#undef __field
#define __field(type, item)

#undef __field_ext
#define __field_ext(type, item, filter_type)

#undef __field_struct
#define __field_struct(type, item)

#undef __field_struct_ext
#define __field_struct_ext(type, item, filter_type)

#undef __array
#define __array(type, item, len)

#undef __dynamic_array
#define __dynamic_array(type, item, len)        u32 item;

#undef __string
#define __string(item, src) __dynamic_array(char, item, -1)

#undef __bitmask
#define __bitmask(item, nr_bits) __dynamic_array(unsigned long, item, -1)

#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)  \
        struct trace_event_data_offsets_##call {                        \
                tstruct;                                                \
        };

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, name, proto, args)
```

以`sched_switch`为例，展开的结果是:

```C
struct trace_event_data_offsets_sched_switch {
    //empty
};
```

没错，这里展开了个寂寞，因为`sched_switch`没有dynamic字段。

### 第5次展开 (3.3.6)

```C
#undef __entry
#define __entry field

#undef TP_printk
#define TP_printk(fmt, args...) fmt "\n", args

...

#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)  \
static notrace enum print_line_t                                        \
trace_raw_output_##call(struct trace_iterator *iter, int flags,         \
                        struct trace_event *trace_event)                \
{                                                                       \
        struct trace_seq *s = &iter->seq;                               \
        struct trace_seq __maybe_unused *p = &iter->tmp_seq;            \
        struct trace_event_raw_##call *field;                           \
        int ret;                                                        \
                                                                        \
        field = (typeof(field))iter->ent;                               \
                                                                        \
        ret = trace_raw_output_prep(iter, trace_event);                 \
        if (ret != TRACE_TYPE_HANDLED)                                  \
                return ret;                                             \
                                                                        \
        trace_seq_printf(s, print);                                     \
                                                                        \
        return trace_handle_return(s);                                  \
}                                                                       \
static struct trace_event_functions trace_event_type_funcs_##call = {   \
        .trace                  = trace_raw_output_##call,              \
};
```

展开为：

```C
static notrace enum print_line_t
trace_raw_output_sched_switch(struct trace_iterator *iter, int flags,
            struct trace_event *trace_event)
{
    struct trace_seq *s = &iter->seq;
    struct trace_seq __maybe_unused *p = &iter->tmp_seq;
    struct trace_event_raw_sched_switch *field;
    int ret;

    field = (typeof(field))iter->ent;

    ret = trace_raw_output_prep(iter, trace_event);
    if (ret != TRACE_TYPE_HANDLED)
        return ret;

    trace_seq_printf(s, print);

    return trace_handle_return(s);
}
static struct trace_event_functions trace_event_type_funcs_sched_switch = {
    .trace            = trace_raw_output_sched_switch,
};
```

这一步定义的函数`trace_raw_output_sched_switch()`用于print `struct trace_event_raw_sched_switch`结构体，即读ftrace ring buffer中的entry结构体，以可读的格式写入用户态output buffer；这个结构体应该由probe创建，存在于ftrace ring buffer中；本函数把它打印成可读形式。注意，这里有一些“奇技淫巧”:

- `field`好像是一个没用的变量；实际上，注意前面有`#define __entry field`，所以，只要使用`__entry`就是使用`field`；
- 可是`__entry`也没有被使用啊！不然。首先`trace_seq_printf(s, print);`中的`print`是第3.2节中`sched_switch`定义中的`TP_printk`。这里`TP_printk`的定义是，把它的`fmt`和`args`取出来。最终`trace_seq_printf`语句变成:

```C
trace_seq_printf(s, "prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> next_comm=%s next_pid=%d next_prio=%d",
        __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,

        (__entry->prev_state & (TASK_REPORT_MAX - 1)) ?
          __print_flags(__entry->prev_state & (TASK_REPORT_MAX - 1), "|",
                        { TASK_INTERRUPTIBLE, "S" },
                        { TASK_UNINTERRUPTIBLE, "D" },
                        { __TASK_STOPPED, "T" },
                        { __TASK_TRACED, "t" },
                        { EXIT_DEAD, "X" },
                        { EXIT_ZOMBIE, "Z" },
                        { TASK_PARKED, "P" },
                        { TASK_DEAD, "I" }) :
          "R",

        __entry->prev_state & TASK_REPORT_MAX ? "+" : "",
        __entry->next_comm, __entry->next_pid, __entry->next_prio)
```

### 第6次展开 (3.3.7)

定义一个函数，ftrace框架使用，不展开。

### 第7次展开 (3.3.8)

定义一个函数，用于计算各个dynamic字段的length和offset；`sched_switch`的`TP_printk`中没有dynamic字段，略过；

### 第8次展开 (3.3.9)

```C
#undef __entry
#define __entry entry

#undef __field
#define __field(type, item)

#undef __field_struct
#define __field_struct(type, item)

#undef __array
#define __array(type, item, len)

...

#undef __assign_bitmask
#define __assign_bitmask(dst, src, nr_bits)                                     \
        memcpy(__get_bitmask(dst), (src), __bitmask_size_in_bytes(nr_bits))

#undef TP_fast_assign
#define TP_fast_assign(args...) args

#undef __perf_count
#define __perf_count(c) (c)

#undef __perf_task
#define __perf_task(t)  (t)

#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)  \
                                                                        \
static notrace void                                                     \
trace_event_raw_event_##call(void *__data, proto)                       \
{                                                                       \
        struct trace_event_file *trace_file = __data;                   \
        struct trace_event_data_offsets_##call __maybe_unused __data_offsets;\
        struct trace_event_buffer fbuffer;                              \
        struct trace_event_raw_##call *entry;                           \
        int __data_size;                                                \
                                                                        \
        if (trace_trigger_soft_disabled(trace_file))                    \
                return;                                                 \
                                                                        \
        __data_size = trace_event_get_offsets_##call(&__data_offsets, args); \
                                                                        \
        entry = trace_event_buffer_reserve(&fbuffer, trace_file,        \
                                 sizeof(*entry) + __data_size);         \
                                                                        \
        if (!entry)                                                     \
                return;                                                 \
                                                                        \
        tstruct                                                         \
                                                                        \
        { assign; }                                                     \
                                                                        \
        trace_event_buffer_commit(&fbuffer);                            \
}
```

展开为：

```C
static notrace void
trace_event_raw_event_sched_switch(void *__data, bool preempt, struct task_struct *prev, struct task_struct *next)
{
    struct trace_event_file *trace_file = __data;
    struct trace_event_data_offsets_sched_switch __maybe_unused __data_offsets;\
    struct trace_event_buffer fbuffer;
    struct trace_event_raw_sched_switch *entry;
    int __data_size;

    if (trace_trigger_soft_disabled(trace_file))
        return;

    __data_size = trace_event_get_offsets_sched_switch(&__data_offsets, args);

    entry = trace_event_buffer_reserve(&fbuffer, trace_file,
                 sizeof(*entry) + __data_size);

    if (!entry)
        return;

    /* tstruct 展开为空，因为没有dynamic字段*/

    /*{ assign }展开为*/
    {
        memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
        __entry->prev_pid       = prev->pid;
        __entry->prev_prio      = prev->prio;
        __entry->prev_state     = __trace_sched_switch_state(preempt, prev);
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->next_pid       = next->pid;
        __entry->next_prio      = next->prio;
        /* XXX SCHED_DEADLINE */
    }

    trace_event_buffer_commit(&fbuffer);
}
```

这就是我们心心念念的probe函数，用了和前面第5次展开一样的“奇技淫巧”，不再赘述。如第3.1节解释Param-4和Param-5时所述：

- `trace_event_buffer_reserve`: 在ftrace ring buffer上分配entry(`trace_event_raw_sched_switch`类型的结构体);
- `assign`: 把入参中的有用信息保存到`entry`；
- `trace_event_buffer_commit`: commit entry到ftrace ring buffer;

另外: `trace_event_get_offsets_sched_switch`用于计算dynamic字段的长度；`sched_switch`没有dynamic字段，不提。

### 第9次展开 (3.3.10)

```C
#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)  \
_TRACE_PERF_PROTO(call, PARAMS(proto));                                 \
static char print_fmt_##call[] = print;                                 \
static struct trace_event_class __used __refdata event_class_##call = { \
        .system                 = TRACE_SYSTEM_STRING,                  \
        .fields_array           = trace_event_fields_##call,            \
        .fields                 = LIST_HEAD_INIT(event_class_##call.fields),\
        .raw_init               = trace_event_raw_init,                 \
        .probe                  = trace_event_raw_event_##call,         \
        .reg                    = trace_event_reg,                      \
        _TRACE_PERF_INIT(call)                                          \
};

#undef DEFINE_EVENT
#define DEFINE_EVENT(template, call, proto, args)                       \
                                                                        \
static struct trace_event_call __used event_##call = {                  \
        .class                  = &event_class_##template,              \
        {                                                               \
                .tp                     = &__tracepoint_##call,         \
        },                                                              \
        .event.funcs            = &trace_event_type_funcs_##template,   \
        .print_fmt              = print_fmt_##template,                 \
        .flags                  = TRACE_EVENT_FL_TRACEPOINT,            \
};                                                                      \
static struct trace_event_call __used                                   \
__section("_ftrace_events") *__event_##call = &event_##call

```

展开为:

```C
static char print_fmt_sched_switch[] = print;

static struct trace_event_class __used __refdata event_class_sched_switch = {
    .system            = TRACE_SYSTEM_STRING,
    .fields_array      = trace_event_fields_sched_switch,
    .fields            = LIST_HEAD_INIT(event_class_sched_switch.fields),
    .raw_init          = trace_event_raw_init,
    .probe             = trace_event_raw_event_sched_switch,
    .reg               = trace_event_reg,
    _TRACE_PERF_INIT(call)
};

static struct trace_event_call __used event_sched_switch = {
    .class            = &event_class_sched_switch,
    {
        .tp            = &__tracepoint_sched_switch,
    },
    .event.funcs        = &trace_event_type_funcs_sched_switch,
    .print_fmt        = print_fmt_sched_switch,
    .flags            = TRACE_EVENT_FL_TRACEPOINT,
};

static struct trace_event_call __used __section("_ftrace_events") *__event_sched_switch = &event_sched_switch;
```

这里初始化第3次展开得到的`event_class_sched_switch`和`event_sched_switch`。其中最重要的是：

- `event_sched_switch.class`指向`event_class_sched_switch`；
- `event_class_sched_switch.probe`指向第8次展开得到的probe函数；
- `event_sched_switch.tp`指向`struct tracepoint`类型的对象`__tracepoint_sched_switch`；这是第1次展开时定义的。详情在第2节介绍了，只是那时没有以`sched_switch`为例子，见第2节`__DECLARE_TRACE`。

```C
struct tracepoint {
        const char *name;               /* Tracepoint name */
        struct static_key key;          /*用于确定是否enabled*/
        struct static_call_key *static_call_key;
        void *static_call_tramp;
        void *iterator;
        int (*regfunc)(void);                /*在add probe之前调用*/
        void (*unregfunc)(void);             /*在remove probe之后调用?*/
        struct tracepoint_func __rcu *funcs; /*probe函数, 允许有多个*/
};
```

如前所述，`struct trace_event_call event_sched_switch`是ftrace event的入口，它的指针被存放在`_ftrace_events`段。系统启动阶段，ftrace初始化时，扫描`_ftrace_events`段，对于其中的每一个`struct trace_event_call`调用`event_init`进行初始化。

### 回顾 (3.3.11)

回顾第3.2节的猜测:

- hook函数调用probe函数，它们的prototype一样。真实情况是：它们虽然不相同，也非常类似。hook函数是`__DECLARE_TRACE`中的`trace_##name`展开的，probe函数见第8次展开：

```C
void trace_sched_switch(bool preempt, struct task_struct *prev, struct task_struct *next);
void trace_event_raw_event_sched_switch(void *__data, bool preempt, struct task_struct *prev, struct task_struct *next);
```
- probe函数在ftrace ring buffer中构造一个entry，类型是`TP_STRUCT__entry`：probe函数确实在ftrace ring buffer中构造一个entry，类型是第3次展开得到的`struct trace_event_raw_sched_switch`；`TP_STRUCT__entry`是其中的一些字段。大体正确。
- probe函数调用`assign`给entry赋值：是的，probe函数`trace_event_raw_event_sched_switch`中包含`assign`中的语句。
- ftrace调用`TP_printk`显示trace到的信息：是的，见第5次展开的`trace_raw_output_sched_switch`函数：print entry (即读ftrace ring buffer中的entry，以可读的格式写入用户态output buffer);

第1次和第2次展开，等价于与旧方式`DECLARE_TRACE/DEFINE_TRACE`；第3次定义了entry的类型；第5次展开定义了打印entry的函数；第8次定义probe函数；第9次把以上所有定义汇集到入口结构体`event_sched_switch`。第4次是处理dynamic字段的，第6和第7次是fatrace框架相关的一些细节，忽略。

最终，得到的是这样一个结构：

{% asset_img trace_event.jpeg TableTreeFormat %}

### ftrace init (3.3.12)

Ftrace初始化是一个复杂的过程，只看和tracepoint相关部分：

```C
start_kernel()
    trace_init()
        trace_event_init()
	    event_trace_enable()
	    {
                ...

                for_each_event(iter, __start_ftrace_events, __stop_ftrace_events) {

                  call = *iter;
                  ret = event_init(call);
                  if (!ret)
                    list_add(&call->list, &ftrace_events);
                }
                
                ...

                __trace_early_add_events(tr);
	    }
```

在`event_trace_enable()`中：

- 扫描`_ftrace_events`段，对其中的每个`struct trace_event_call`结构体`call`：1. 调用`event_init(call)`；2. 把`call`添加到`ftrace_events`(全局变量，ftrace的所有event的列表)；
- `__trace_early_add_events`函数里再次扫描`_ftrace_events`段，为每个`call`创建`struct trace_event_file`文件。这个文件应该是`/sys/kernel/debug/tracing/events/sched/sched_switch/enable`；看probe函数(第8次展开)，通过这个文件判断event是否soft disabled。第3.2节中演练也是`echo 1`到这个文件来enable event;

### enable event (3.3.13)

第2节说，要通`register_trace_##name`来把probe关联到hook上；且在第2节`__DECLARE_TRACE`中可以看到`register_trace_##name`调用`tracepoint_probe_register`。所以，我们有理由猜测，要enable event，要么调用`register_trace_##name`，要么调用`tracepoint_probe_register`。

在第3.2节演练时，通过`echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable`来enable event。Ftrace通过一个虚拟的filesystem和user交互：

```C
static const struct file_operations ftrace_enable_fops = {
        .open = tracing_open_generic,
        .read = event_enable_read,
        .write = event_enable_write,
        .llseek = default_llseek,
};
```

所以`echo 1`触发的是`event_enable_write`函数。我们看看它是否会调用到`register_trace_##name`或`tracepoint_probe_register`。

```C
static ssize_t
event_enable_write(struct file *filp, const char __user *ubuf, size_t cnt,
                   loff_t *ppos)
{
        struct trace_event_file *file;
        unsigned long val;
        int ret;

        ret = kstrtoul_from_user(ubuf, cnt, 10, &val);
        if (ret)
                return ret;

        ret = tracing_update_buffers();
        if (ret < 0)
                return ret;

        switch (val) {
        case 0:
        case 1:
                ret = -ENODEV;
                mutex_lock(&event_mutex);
                file = event_file_data(filp);
                if (likely(file))
                        ret = ftrace_event_enable_disable(file, val);
                mutex_unlock(&event_mutex);
                break;

        default:
                return -EINVAL;
        }

        *ppos += cnt;

        return ret ? ret : cnt;
}
```

首先，这个函数是非常规范的文件系统write函数：入参有文件指针`filp`，数据buffer `ubuf`, 数据长度`cnt`，以及写入到文件中的位置`ppos`；这里数据应该"1"，位置应该是0；文件指针`filp`是不是`/sys/kernel/debug/tracing/events/sched/sched_switch/enable`呢？应该是的，`event_file_data`定义如下：

```C
static inline void *event_file_data(struct file *filp)
{
        return READ_ONCE(file_inode(filp)->i_private);
}
```

所以，`filp`的`inode`的`i_private`就是`/sys/kernel/debug/tracing/events/sched/sched_switch/enable`。

当写入的是"0"或"1"时，就会调用`ftrace_event_enable_disable()`，`val`是"0"或"1"。


```C
static int ftrace_event_enable_disable(struct trace_event_file *file,
                                       int enable)
{
        return __ftrace_event_enable_disable(file, enable, 0);
}


static int __ftrace_event_enable_disable(struct trace_event_file *file,
                                         int enable, int soft_disable)
{
        struct trace_event_call *call = file->event_call;
        struct trace_array *tr = file->tr;
        unsigned long file_flags = file->flags;
        int ret = 0;
        int disable;

        switch (enable) {
        case 0:

                ......

                break;
        case 1:
                if (!(file->flags & EVENT_FILE_FL_ENABLED)) {

                        ......

                        ret = call->class->reg(call, TRACE_REG_REGISTER, file);

                        ......

                }
                break;
        }

        ......

        return ret;
}
```

看第9次展开知道，`call->class->reg`是`trace_event_reg`函数。

```C
int trace_event_reg(struct trace_event_call *call,
                    enum trace_reg type, void *data)
{
        struct trace_event_file *file = data;

        WARN_ON(!(call->flags & TRACE_EVENT_FL_TRACEPOINT));
        switch (type) {
        case TRACE_REG_REGISTER:
                return tracepoint_probe_register(call->tp,
                                                 call->class->probe,
                                                 file);
        case TRACE_REG_UNREGISTER:
                tracepoint_probe_unregister(call->tp,
                                            call->class->probe,
                                            file);
                return 0;

#ifdef CONFIG_PERF_EVENTS
        case TRACE_REG_PERF_REGISTER:
                return tracepoint_probe_register(call->tp,
                                                 call->class->perf_probe,
                                                 call);
        case TRACE_REG_PERF_UNREGISTER:
                tracepoint_probe_unregister(call->tp,
                                            call->class->perf_probe,
                                            call);
                return 0;
        case TRACE_REG_PERF_OPEN:
        case TRACE_REG_PERF_CLOSE:
        case TRACE_REG_PERF_ADD:
        case TRACE_REG_PERF_DEL:
                return 0;
#endif
        }
        return 0;
}
EXPORT_SYMBOL_GPL(trace_event_reg);
```

符合前面的猜测：要enable event，要么调用`register_trace_##name`，要么调用`tracepoint_probe_register`；这里调用了后者。

### 小结 (3.3.14)

至此，搞清楚了tracepoint和ftrace的关系：

- tracepoint是静态的hook(内核代码中的`trace_##name`)加上动态register/unregister的probe；
- 使用`DECLARE_TRACE/DEFINE_TRACE`可以创建一个tracepoint；这是内核代码开发者的事。想要使用它，需要调用开发一个内核模块，实现probe并register；
- 上述使用不是很方便，所以，随着系统演进，创建tracepoint时，自动附带一个probe，由ftrace来register，即tracepoint自动作为ftrace的event-source；`TRACE_EVENT`宏定义了很多复杂的数据结构和函数来实现这个目的。
- 当然，tracepoint和ftrace不是绑定的，还可以像以前那样使用tracepoint；另外，tracepoint也可以作为其它tracer(如perf, LTTng, SystemTap)的event-source；

# 作为perf的event-source (4)

根据tracepoint的原理，perf要使用它，也是要实现一个probe函数并register。其实，从第3.3节中第3次展开已经看到一些端倪:

```C
struct trace_event_class {
        const char              *system;
        void                    *probe;
#ifdef CONFIG_PERF_EVENTS
        void                    *perf_probe;
#endif
        int                     (*reg)(struct trace_event_call *event,
                                       enum trace_reg type, void *data);
        struct trace_event_fields *fields_array;
        struct list_head        *(*get_fields)(struct trace_event_call *);
        struct list_head        fields;
        int                     (*raw_init)(struct trace_event_call *);
};
```

当配置了perf时，`struct trace_event_class event_class_sched_switch`就定义了一个`perf_probe`指针。而第9次展开是这样的：


```C

......

static struct trace_event_class __used __refdata event_class_sched_switch = {
    .system            = TRACE_SYSTEM_STRING,
    .fields_array      = trace_event_fields_sched_switch,
    .fields            = LIST_HEAD_INIT(event_class_sched_switch.fields),
    .raw_init          = trace_event_raw_init,
    .probe             = trace_event_raw_event_sched_switch,
    .reg               = trace_event_reg,
    _TRACE_PERF_INIT(call)
};

......

```

其中`_TRACE_PERF_INIT`被忽略了，它的定义是：

```C
#define _TRACE_PERF_INIT(call)                   \
        .perf_probe             = perf_trace_##call,

```

所以，完全展开是：

```C

......

static struct trace_event_class __used __refdata event_class_sched_switch = {
    .system            = TRACE_SYSTEM_STRING,
    .fields_array      = trace_event_fields_sched_switch,
    .fields            = LIST_HEAD_INIT(event_class_sched_switch.fields),
    .raw_init          = trace_event_raw_init,
    .probe             = trace_event_raw_event_sched_switch,
    .reg               = trace_event_reg,
    .perf_probe        = perf_trace_sched_switch,
};

......

```

我们要找到`perf_trace_sched_switch`，它就是为perf提供的probe。第3.3节中，为了支持ftrace，`TRACE_EVENT`被展开7次，本节还是熟悉的配方，为了支持perf，`TRACE_EVENT`在include/trace/perf.h(linux-5.10.161)继续展开:

```C
#undef __entry
#define __entry entry

#undef __get_dynamic_array
#define __get_dynamic_array(field)      \
                ((void *)__entry + (__entry->__data_loc_##field & 0xffff))

#undef __get_dynamic_array_len
#define __get_dynamic_array_len(field)  \
                ((__entry->__data_loc_##field >> 16) & 0xffff)

#undef __get_str
#define __get_str(field) ((char *)__get_dynamic_array(field))

#undef __get_bitmask
#define __get_bitmask(field) (char *)__get_dynamic_array(field)

#undef __perf_count
#define __perf_count(c) (__count = (c))

#undef __perf_task
#define __perf_task(t)  (__task = (t))

#undef DECLARE_EVENT_CLASS
#define DECLARE_EVENT_CLASS(call, proto, args, tstruct, assign, print)  \
static notrace void                                                     \
perf_trace_##call(void *__data, proto)                                  \
{                                                                       \
        struct trace_event_call *event_call = __data;                   \
        struct trace_event_data_offsets_##call __maybe_unused __data_offsets;\
        struct trace_event_raw_##call *entry;                           \
        struct pt_regs *__regs;                                         \
        u64 __count = 1;                                                \
        struct task_struct *__task = NULL;                              \
        struct hlist_head *head;                                        \
        int __entry_size;                                               \
        int __data_size;                                                \
        int rctx;                                                       \
                                                                        \
        __data_size = trace_event_get_offsets_##call(&__data_offsets, args); \
                                                                        \
        head = this_cpu_ptr(event_call->perf_events);                   \
        if (!bpf_prog_array_valid(event_call) &&                        \
            __builtin_constant_p(!__task) && !__task &&                 \
            hlist_empty(head))                                          \
                return;                                                 \
                                                                        \
        __entry_size = ALIGN(__data_size + sizeof(*entry) + sizeof(u32),\
                             sizeof(u64));                              \
        __entry_size -= sizeof(u32);                                    \
                                                                        \
        entry = perf_trace_buf_alloc(__entry_size, &__regs, &rctx);     \
        if (!entry)                                                     \
                return;                                                 \
                                                                        \
        perf_fetch_caller_regs(__regs);                                 \
                                                                        \
        tstruct                                                         \
                                                                        \
        { assign; }                                                     \
                                                                        \
        perf_trace_run_bpf_submit(entry, __entry_size, rctx,            \
                                  event_call, __count, __regs,          \
                                  head, __task);                        \
}
```

展开为:

```C
static notrace void
perf_trace_sched_switch(void *__data, bool preempt, struct task_struct *prev, struct task_struct *next)
{
    struct trace_event_call *event_call = __data;
    struct trace_event_data_offsets_sched_switch __maybe_unused __data_offsets;\
    struct trace_event_raw_sched_switch *entry;
    struct pt_regs *__regs;
    u64 __count = 1;
    struct task_struct *__task = NULL;
    struct hlist_head *head;
    int __entry_size;
    int __data_size;
    int rctx;

    __data_size = trace_event_get_offsets_sched_switch(&__data_offsets, args);

    head = this_cpu_ptr(event_call->perf_events);
    if (!bpf_prog_array_valid(event_call) &&
        __builtin_constant_p(!__task) && !__task &&
        hlist_empty(head))
            return;

    __entry_size = ALIGN(__data_size + sizeof(*entry) + sizeof(u32),\
                         sizeof(u64));
    __entry_size -= sizeof(u32);

    entry = perf_trace_buf_alloc(__entry_size, &__regs, &rctx);
    if (!entry)
            return;

    perf_fetch_caller_regs(__regs);

    /* tstruct 展开为空，因为没有dynamic字段*/

    /*{ assign }展开为*/
    {
        memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
        __entry->prev_pid       = prev->pid;
        __entry->prev_prio      = prev->prio;
        __entry->prev_state     = __trace_sched_switch_state(preempt, prev);
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->next_pid       = next->pid;
        __entry->next_prio      = next->prio;
        /* XXX SCHED_DEADLINE */
    }


    perf_trace_run_bpf_submit(entry, __entry_size, rctx,
                              event_call, __count, __regs,
                              head, __task);
}
```

不出所料，我们找到了`perf_trace_sched_switch`。看上去是不是和第3.3节第8次展开很像？是的，**它就是perf版本的probe函数**，所做的事情也非常类似:

-  在perf ring buffer上分配entry(`trace_event_raw_sched_switch`类型的结构体)，而不是在ftrace ring buffer上。
- `assign`: 把入参中的有用信息保存到`entry`；这一点是一模一样的；
- `perf_trace_run_bpf_submit`: commit entry;

同样，`trace_event_get_offsets_sched_switch`计算dynamic字段的长度；`sched_switch`没有dynamic字段，不提；

有了perf的probe函数，想必它也是enable的时候register的。其实第3.3节中"enable event"小节中，`trace_event_reg`函数已经处理了：

```C
int trace_event_reg(struct trace_event_call *call,
                    enum trace_reg type, void *data)
{
        ......

        switch (type) {

        ......

#ifdef CONFIG_PERF_EVENTS
        case TRACE_REG_PERF_REGISTER:
                return tracepoint_probe_register(call->tp,
                                                 call->class->perf_probe,
                                                 call);
        case TRACE_REG_PERF_UNREGISTER:
                tracepoint_probe_unregister(call->tp,
                                            call->class->perf_probe,
                                            call);
                return 0;
        case TRACE_REG_PERF_OPEN:
        case TRACE_REG_PERF_CLOSE:
        case TRACE_REG_PERF_ADD:
        case TRACE_REG_PERF_DEL:
                return 0;
#endif
        }
        return 0;
}
EXPORT_SYMBOL_GPL(trace_event_reg);
```

用户态程序通过syscall `perf_event_open()`来enable perf event；这个函数定义在`kernel/events/core.c`(linux-5.10.161)中；`perf_event_open()`调用`pmu->event_init`来初始化event；对于tracepoint类型的event来说，`pmu`就是`struct pmu perf_tracepoint`；其`event_init`指向`perf_tp_event_init()`函数。

```C
perf_tp_event_init()
    perf_trace_init()
        perf_trace_event_init()
            perf_trace_event_reg()
            {
                tp_event->class->reg(tp_event, TRACE_REG_PERF_REGISTER, NULL);
            }
```

其中`tp_event->class->reg`就是函数`trace_event_reg`，传入的`type`是`TRACE_REG_PERF_REGISTER`，所以会调用: `tracepoint_probe_register(..., call->class->perf_probe, ...)`，即register probe `perf_trace_sched_switch`。

总之，perf使用tracepoint的方式，也是定义probe并register它。

# 作为eBPF的event-source (5)

事先说明，eBPF是通过perf来使用tracepoint的，而没有定义自己的“ebpf probe”函数并register。魔法就在于perf probe函数`perf_trace_sched_switch`的最后一句：

```C
static notrace void
perf_trace_sched_switch(void *__data, bool preempt, struct task_struct *prev, struct task_struct *next)
{

    ......

    perf_trace_run_bpf_submit(entry, __entry_size, rctx,
                              event_call, __count, __regs,
                              head, __task);
}
```

这里要提交entry，但不一定提交到perf的ring buffer：

```C
# kernel/events/core.c
void perf_trace_run_bpf_submit(void *raw_data, int size, int rctx,
                               struct trace_event_call *call, u64 count,
                               struct pt_regs *regs, struct hlist_head *head,
                               struct task_struct *task)
{
        if (bpf_prog_array_valid(call)) {
                *(struct pt_regs **)raw_data = regs;
                if (!trace_call_bpf(call, raw_data) || hlist_empty(head)) {
                        perf_swevent_put_recursion_context(rctx);
                        return;
                }
        }
        perf_tp_event(call->event.type, count, raw_data, size, regs, head,
                      rctx, task);
}
EXPORT_SYMBOL_GPL(perf_trace_run_bpf_submit);
```

当`bpf_prog_array_valid()`返回false时，才会调用`perf_tp_event()`提交到perf的ring buffer；这属于上一节的内容。

本节我们看`bpf_prog_array_valid()`为true时的情形。它什么时候返回true呢？

```C
#ifdef CONFIG_PERF_EVENTS
static inline bool bpf_prog_array_valid(struct trace_event_call *call)
{
        return !!READ_ONCE(call->prog_array);
}
#endif
```

回顾第3.3节第3次展开的`struct trace_event_call`，里面就有`prog_array`：

```C
struct trace_event_call {

        ......

#ifdef CONFIG_PERF_EVENTS
        int                             perf_refcount;
        struct hlist_head __percpu      *perf_events;
        struct bpf_prog_array __rcu     *prog_array;   /*数组每个元素是一个eBPF program，最多BPF_TRACE_MAX_PROGS=64个*/

        int     (*perf_perm)(struct trace_event_call *,
                             struct perf_event *);
#endif
};
```

当syscall`ioctl(event_fd, PERF_EVENT_IOC_SET_BPF, prog_fd)`被调用时，经过`perf_event_set_bpf_prog`调用`perf_event_attach_bpf_prog`安装一个eBPF program (program之前已经被load到内核并返回了`prog_fd`，这里通过`prog_fd`指定该program)；见`kernel/events/core.c`(linux-5.10.161)：

```C
static long _perf_ioctl(struct perf_event *event, unsigned int cmd, unsigned long arg)
{
        void (*func)(struct perf_event *);
        u32 flags = arg;

        switch (cmd) {

        ......

        case PERF_EVENT_IOC_SET_BPF:
                return perf_event_set_bpf_prog(event, arg);

        ......

        }

        ......

        return 0;
}

static int perf_event_set_bpf_prog(struct perf_event *event, u32 prog_fd)
{
        struct bpf_prog *prog;
        prog = bpf_prog_get(prog_fd);

        ......

        ret = perf_event_attach_bpf_prog(event, prog);

        ......

        return ret;
}
```

函数`perf_event_attach_bpf_prog`在`kernel/trace/bpf_trace.c`(linux-5.10.161)中：

```C
#define BPF_TRACE_MAX_PROGS 64

int perf_event_attach_bpf_prog(struct perf_event *event,
                               struct bpf_prog *prog)
{
        struct bpf_prog_array *old_array;
        struct bpf_prog_array *new_array;
        int ret = -EEXIST;

        ......

        mutex_lock(&bpf_event_mutex);

        if (event->prog)
                goto unlock;

        old_array = bpf_event_rcu_dereference(event->tp_event->prog_array);
        if (old_array &&
            bpf_prog_array_length(old_array) >= BPF_TRACE_MAX_PROGS) {
                ret = -E2BIG;
                goto unlock;
        }

        ret = bpf_prog_array_copy(old_array, NULL, prog, &new_array);
        if (ret < 0)
                goto unlock;

        /* set the new array to event->tp_event and set event->prog */
        event->prog = prog;
        rcu_assign_pointer(event->tp_event->prog_array, new_array);
        bpf_prog_array_free(old_array);

unlock:
        mutex_unlock(&bpf_event_mutex);
        return ret;
}
```

它看上去有点绕，其实是这样的：`event->tp_event`就是`struct trace_event_call`类型指针；如前所述，它有一个`prog_array`成员；`bpf_prog_array_valid()`就是判断这个成员。这里就是:

- `old_array` = `event->tp_event->prog_array`; 原来可能为空，也可能不空；
- `new_array` = `old_array + prog`；`bpf_prog_array_copy`的原型是`bpf_prog_array_copy(old_array, exclude_prog, include_prog, new_array);`
- `event->tp_event->prog_array = new_array`;

所以，经过attach，`bpf_prog_array_valid()`就会返回true;

回到到前面`perf_trace_run_bpf_submit()`函数，当用户attach了eBPF program，就会通过`trace_call_bpf`执行program；

```C
# kernel/events/core.c
void perf_trace_run_bpf_submit(void *raw_data, int size, int rctx,
                               struct trace_event_call *call, u64 count,
                               struct pt_regs *regs, struct hlist_head *head,
                               struct task_struct *task)
{
        if (bpf_prog_array_valid(call)) {
                *(struct pt_regs **)raw_data = regs;
                if (!trace_call_bpf(call, raw_data) || hlist_empty(head)) {
                        perf_swevent_put_recursion_context(rctx);
                        return;
                }
        }
        perf_tp_event(call->event.type, count, raw_data, size, regs, head,
                      rctx, task);
}

# kernel/trace/bpf_trace.c
unsigned int trace_call_bpf(struct trace_event_call *call, void *ctx)
{
        cant_sleep();

        ......

        ret = BPF_PROG_RUN_ARRAY_CHECK(call->prog_array, ctx, BPF_PROG_RUN);

        ......

        return ret;
}

# include/linux/bpf.h
#define BPF_PROG_RUN_ARRAY_CHECK(array, ctx, func)      \
        __BPF_PROG_RUN_ARRAY(array, ctx, func, true, false)

#define __BPF_PROG_RUN_ARRAY(array, ctx, func, check_non_null, set_cg_storage) \
        ({                                              \
                struct bpf_prog_array_item *_item;      \
                struct bpf_prog *_prog;                 \
                struct bpf_prog_array *_array;          \
                u32 _ret = 1;                           \
                migrate_disable();                      \
                rcu_read_lock();                        \
                _array = rcu_dereference(array);        \
                if (unlikely(check_non_null && !_array))\
                        goto _out;                      \
                _item = &_array->items[0];              \
                while ((_prog = READ_ONCE(_item->prog))) {              \
                        if (!set_cg_storage) {                  \
                                _ret &= func(_prog, ctx);       \
                        } else {                                \
                                if (unlikely(bpf_cgroup_storage_set(_item->cgroup_storage)))    \
                                        break;                  \
                                _ret &= func(_prog, ctx);       \
                                bpf_cgroup_storage_unset();     \
                        }                               \
                        _item++;                        \
                }                                       \
_out:                                                   \
                rcu_read_unlock();                      \
                migrate_enable();                       \
                _ret;                                   \
         })
```

这里逐一执行eBPF program。

总接来说：eBPF使用tracepoint做event-source，绝大多数工作都是通过perf完成的，即probe函数做的那些事，分配entry并赋值等。只是最后不提交到perf的ring buffer里，而是触发eBPF program的执行(存储到eBPF的table/storage中?)。

# 总结 (6)

Linux tracepint是静态的事件源，内核代码中有很多`trace_##subsys_##event()`调用。这些调用就是hook，正常情况下是空的。若想使用它，就给它关联(注册)probe函数，这样当内核运行到hook处就会调用probe。本文以tracepoint为焦点，首先介绍了它的创建与使用，即通过`DECLARE_TRACE/DEFINE_TRACE`创建，通过`tracepoint_probe_register`注册probe来使用。其实这已经是完整的tracepoint了。只是使用起来有门槛。

为了方便使用，又把它作为ftrace以及perf的是event-source，这样使用者不必自己开发并注册probe函数，而是通过ftrace或者perf来使用它。内核代码中预置了ftrace以及perf版本的probe，使用者通过ftrace和perf的接口就可以注册并触发。宏`TRACE_EVENT`就是在调用`DECLARE_TRACE/DEFINE_TRACE`创建tracepoint之后，预置ftrace以及perf版本的probe函数(以及一些配套的数据结构)。在perf的基础上，eBPF也可以使用tracepoint。

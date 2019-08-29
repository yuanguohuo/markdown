---
title: Java的Error和Exception
date: 2019-08-23 16:24:21
tags: [exception,checked,unchecked]
categories: java 
---

简单记录一下Java的`Error`和`Exception`的区别，以及checked和unchecked的区别。

<!-- more -->

# Throwable (1)

![Throwable](throwable.jpg)

# Error和Exception (2)

`Error`类主要代表程序运行的环境出错。例如`OutOfMemoryError`表示JVM内存耗尽；`StackOverflowError`表示栈溢出。不能用`try-catch`来处理`Error`(即使能用catch住，也无法recover)。如上图所示，`Error`都是unchecked(见下文)。

与`Error`不同，`Exception`类主要代表程序本身的错误，可以使用`try-catch`处理。`Exceptio`n分为两种：checked和unchecked。

# checked和unchecked (3)

- checked：编译器在编译阶段会检查这类异常，它强迫程序员要么用`try-catch`处理，要么使用`throws`声明抛出。这也是"checked"这个词的意义。Checked异常一般是可以被recover的。

- unchecked：编译器在编译阶段不会检查这类错误。它包含`Error`和`RuntimeException`(及其子类)。1.`Error`主要代表程序运行的环境出错(比如`OutOfMemoryError`和`StackOverflowError`)，所以只有运行时才可能发生。2.`RuntimeException`主要表示编程错误，例如程序运行时发生除零错误会抛出`ArithmeticException`；访问数组越界会抛出`ArrayIndexOutOfBoundsException`，访问空指针会发生`NullPointerException`。这主要是程序员的问题。它们可能发生在任何地方，也可能非常多。假如把它们归入checked类，那么几乎所有的函数都需要catch它们或声明throws它们，使得程序很不简洁。并且，假如你声明throws一个`ArithmeticException`，相当于告诉你的调用者：我这段程序可能抛出除零异常。这不是你应该避免的吗？难道你知道哪里有可能还不检查被除数？另外，unchecked异常也很难被recover：环境出错自不必说；编程错误如何recover呢？比如你调用了一个函数，它抛出了除零异常，或者数组越界。

总之，unchecked错误里，`Error`是环境错误；`RuntimeException`是编程错误；都难以recover。所以，一般来说: 

- 不要抛出`RuntimeException`，也不要继承它定义异常类。
- 一个错误，你的调用者catch到它时，若可能recover（比如IO异常，可以重试），就把它定义为checked；否则，调用者不能recover，则定义为unchecked。

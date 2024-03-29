---
title: Python3 Metaclass 
date: 2024-02-15 13:09:03
tags: [Python3,metaclass]
categories: Python3 
---

简要记录Python3中的metaclass。注意本文是基于new-style class的。Python3中所有class都是new-style class，Python2则不同：Python2.2以前根本不支持new-style class，而从Python2.2开始，虽然支持但需要显示地声明。

# 一切皆对象 (1)

在Python中，一切都是对象，比如: `int`, `3`, 自定义类的实例`foo`，以及自定义类`Foo`，所以下面的语句都打印`True`:

```Python
print(isinstance(3, object))
print(isinstance(int, object))
print(isinstance(foo, object))
print(isinstance(Foo, object))
```

`Foo`是对象，那么它的class是什么呢？答案是metaclass：默认情况下，`Foo`是metaclass `type`的实例，我们也可以自定义metaclass，见第2节。

![figure1](metaclass.png)
<div style="text-align: center;"><em>图1</em></div>

如图所示:

- `foo` is an instance of class `Foo`.
- `Foo` is an instance of the `type` metaclass.
- `type` is also an instance of the `type` metaclass, so it is an instance of itself.


# 自定义metaclass (2)

先看这段代码:

```python
#!/usr/bin/python3

class Meta(type):
    def __init__(self, *args, **kwargs):
        print('Meta.__init__ enter')
        super().__init__(*args, **kwargs)

    def __new__(cls, *args, **kwargs):
        print('Meta.__new__ enter')
        return super().__new__(cls, *args, **kwargs)

    def __call__(self, *args, **kwargs):
        print('Meta.__call__ enter')
        return super().__call__(*args, **kwargs)


class Foo:
    def __init__(self):
        print('Foo.__init__ enter')
        super().__init__()

    def __new__(cls):
        print('Foo.__new__ enter')
        return super().__new__(cls)

    def __call__(self, *args, **kwargs):
        print('Foo.__call__ enter')


class Bar(Foo, metaclass=Meta):
    def __init__(self):
        print('Bar.__init__ enter')
        super().__init__()

    def __new__(cls):
        print('Bar.__new__ enter')
        return super().__new__(cls)

    def __call__(self, *args, **kwargs):
        print('Bar.__call__ enter')


if __name__ == '__main__':
    print('================== main enter ==================')
    foo = Foo()
    bar = Bar()
```
用图形表示：

![figure2](meta-and-inheritance.png)
<div style="text-align: center;"><em>图2</em></div>

当python解析器遇见`Foo()`语句时：

- 调用`Foo`的metaclass的`__call__`，即调用`type.__call__`，**注意：不是`Foo.__call__**.
- `Foo.__call__`调用：
    - `Foo.__new__`
    - `Foo.__init__`

当python解析器遇见`Bar()`语句时：

- 调用`Bar`的metaclass的`__call__`，即调用`Meta.__call__`，**注意：不是`Bar.__call__**.
- `Meta.__call__`调用：
    - `Bar.__new__`
    - `Bar.__init__`

注意区分**object-class的从属关系**和**class-class的继承关系**：

- 找`__call__`时，**沿着object-class的从属关系往上走**。
- 找`__new__`和`__init__`时才进入继承领域，**沿着class-class的继承关系往上走**。

所以，上面代码的输出是：

```
Meta.__new__ enter
Meta.__init__ enter
================== main enter ==================
                     <--- Foo()，这里调用type.__call__
Foo.__new__ enter
Foo.__init__ enter
Meta.__call__ enter  <--- Bar()
Bar.__new__ enter
Foo.__new__ enter    //继承，调用parent-class的__new__
Bar.__init__ enter
Foo.__init__ enter   //继承，调用parent-class的__init__
```

上面重点解释了`foo`和`bar`的创建过程。但不是说**一切皆对象**吗？`Foo`和`Bar`本身是怎么创建的呢？注意`main enter`之前还有两行输出，它们就是创建`Bar`的过程，其实和创建实例的过程一样：

- 先调用`Bar`的class的metaclass的`__call__`：**沿着object-class的从属关系往上走**，`Bar`的class是`Meta`(就是Bar的metaclass)，`Meta`的metaclass是`type`，所以这一步调用`type.__call__`，什么也没有打印。
- `type.__call__`调用：
    - `Meta.__new__`
    - `Meta.__init__`

这里注意class的class就是它的metclass；这应该也是**meta**一词的来源。
另外，`Foo`的的metaclass是`type`，我们没有机会在它里面添加打印语句，所以`Foo`的创建什么都没打印。

# 通过`type`动态创建class (3)

下面两种方式是等价的：

```Python
def plus(self, i):
    self.m = self.m + i
    print(self.m)


def minus(self, i):
    self.m = self.m - i
    print(self.m)


Foo = type('Foo', (), {'add': plus, 'sub': minus})
```

```python
class Foo:
    m = 100

    def add(self, i):
        self.m = self.m + i
        print(self.m)

    def sub(self, i):
        self.m = self.m - i
        print(self.m)
```


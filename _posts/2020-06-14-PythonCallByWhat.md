---
layout: post
title:  "Python Call by What"
date:   2020-06-14
categories: jekyll update
---

以下是几种常见的求值策略（或称调用模型、参数传递惯例、规约策略）在 FOLDOC ([Free On-Line Dictionary Of Computing](http://foldoc.org)) 中的定义：

> Call by value (CBV): An evaluation strategy where arguments are evaluated before the function or procedure is entered. Only the values of the arguments are passed and changes to the arguments within the called procedure have no effect on the actual arguments as seen by the caller.

> Call by reference (CBR): An argument passing convention where the address of an argument variable is passed to a function or procedure, as opposed to passing the value of the argument expression. Execution of the function or procedure may have side-effects on the actual argument as seen by the caller.

> Call by name (CBN): An argument passing convention where argument expressions are passed unevaluated. This is usually implemented by passing a pointer to a thunk - some code which will return the value of the argument and an environment giving the values of its free variables.

CBV 比 CBN 更加高效，但存在无限数据结构或递归函数时也更可能出现不终止的情况。此外，call by need = CBN + graph reduction，常用于函数式语言，可以避免同一表达式的重复求值。

Python 的调用模型是 CBV 还是 CBR 一直争论不休，首先根据 Fredrik Lundh（Python 核心开发者，曾获 2003 年 [Frank Willison Award](https://www.python.org/community/awards/frank-willison/) — Python 社区年度杰出贡献奖）的[文章](http://effbot.org/zone/call-by-object.htm)我们给出结论，Python 采用 [CLU](http://publications.csail.mit.edu/lcs/pubs/pdf/MIT-LCS-TR-561.pdf) 语言 call by sharing（或称 call by object）的调用模型。

### Python Objects

所有的 Python 对象都有：

- 一个唯一的身份标识：一个用 `id(obj)` 返回的整型，在 CPython 中就是 `obj` 的地址（[The Python Language Reference - Data model - Objects, values and types](https://docs.python.org/3/reference/datamodel.html#objects-values-and-types)）。
- 一个类型：用 `type(obj)` 返回。
- 一些内容（或称值、状态）。

身份标识无法更改；类型以类型对象表示，也无法更改（CPython 2.2 及以后在[极少数特殊情形](https://docs.python.org/3/whatsnew/2.2.html#peps-252-and-253-type-and-class-changes)下可以更改）；一些对象允许改变其内容（身份标识和类型不变）。

Python 对象可能有：

- 0 或多个方法（由类型对象提供）
- 0 或多个名称

对象可能没有方法或只有允许读取内容的方法（immutable），也可能有允许改变内容的方法（mutable，time-varying behavior）。

名称并不属于对象本身（对象不知道自己的名称），名称存在于命名空间（模块/实例/函数）中，或者说命名空间是名称关联对象引用的字典。以赋值为例，赋值语句修改命名空间，而非对象：`name = 10` 把名称 `name` 加入局部命名空间，并指向一个整型对象，该整型对象包含值 `10`。继续执行 `name = 20` 覆盖名称 `name`，指向一个整型包含值 `20` 对象，原本的 `10` 对象不受任何影响。

```python
name = []
name.append(1)
```

首先把名称 `name` 加入局部命名空间，指向一个空列表对象，这步会修改命名空间；然后调用该对象的方法修改列表对象的内容，这步不操作命名空间，也不操作整型对象。

区别于赋值语句，属性赋值（attribute assignment）`name.attr` 是方法 `__setattr__`/`__getattr__` 的语法糖；项引用（item reference）`name[index]` 是方法 `__setitem__`/`__getitem__` 的语法糖。

### Call By Sharing (CBS)

Liskov 在 [CLU Reference Manual](https://dl.acm.org/doi/book/10.5555/889832) 中给出的定义：

> We call the argument passing technique _call by sharing_, because the argument objects are shared between the caller and the called routine.  This technique does not correspond to most traditional argument passing techniques (it is similar to argument passing in LISP).  In particular it is not call by value because mutations of arguments performed by the called routine will be visible to the caller. And it is not call by reference because access is not given to the variables of the caller, but merely to certain objects.

即被调用者对参数的修改对调用者而言是可见的，所以不是 CBV；传入的也不是对象的地址值，形参和实参共享同一个对象，而非实参对应对象的值（内容）、形参对应对象的地址，所以也不是 CBR。

注意变量这个概念，变量在多数语言中也是一个对象，它包含值；而在 CLU 和 Python 中，变量只是一个指向对象的名称，这像指针，但和指针的区别是本身不能被其他变量指向。多个变量分享对象，变量不能指向变量，但是对象可以指向对象，如一个实例包含多个组件，这是一种逻辑而非物理上的包含，不同实例也可以指向同一组件对象。对象也可以指向其自身，这样，在没有显式引用类型的情况下也可以定义递归数据类型和共享数据对象了。

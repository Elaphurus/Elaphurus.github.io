---
layout: post
title:  "简单地从类型系统的角度看编程语言"
date:   2020-04-06
categories: jekyll update
---

我们经常会听到这样的讨论，“动态语言不适合大型项目”，“xx 语言不是类型安全的”，诸如此类。但对于这些概念我们真的清楚吗？

一般来说，从类型系统的角度，我们有三个相互独立的概念：

- 强类型/弱类型
- 静态定型/动态定型
- 类型安全/非类型安全

下面我们非形式化地讨论一下这几个概念。

### 强类型/弱类型

强类型语言，即在该编程语言中，没有隐性的数据类型转换。

一个典型的例子如下：

```
a = '2'
b = a - 1
```

在 JavaScript，b 的值为 1；而在 Python 中，则会产生错误 `TypeError: unsupported operand type(s) for -: 'str' and 'int'`。

JavaScript 的顺利执行，归因于编译器（解释器）隐性地完成了类型转换的操作，对应的 Python 代码应该像下面这样：

```Python
a = '2'
b = int(a) - 1
```

但是一个语言是强类型还是弱类型有时是有争议的，因为哪些转换是允许的并没有一个很严格的界定。考虑下面的 Python 代码：

```Python
a = 2
b = a - 1.0
```

其中也存在 int 到 float 的隐性类型转换，但是可以顺利执行。

这样的隐性类型转换往往就是允许的，因为也是符合数学直觉的（减法运算的操作数是数值，而非字符串）。

### 静态定型/动态定型

静态定型，在编译时进行类型检查；动态定型，在运行时进行类型检查。这一概念往往比较清楚，因为静态定型依赖于类型标注，比如前面的例子在 C 语言中就必须写成：

```C
int a = 2;
int b = a - 1;
```

### 类型安全/非类型安全

类型安全，即不会出现逃逸出该编程语言类型规则的情况。这个概念比较抽象，有时会被混淆。所谓逃逸出类型规则，典型的例子是 C 语言的类型双关（type punning）。

```C
bool is_negative(float x) {
    unsigned int *ui = (unsigned int *)&x;
    return *ui & 0x80000000;
}
```

在使用 IEEE 754 浮点数标准的条件下，利用指针类型转换实现类型双关来获取浮点数的符号位做整型计算，可以比以下简单的浮点计算得到更优的性能。

```C
bool is_negative(float x) {
    return x < 0.0;
}
```

注意负零（10000000）时的区别。

类型双关也可以用 union 来实现：

```C
bool is_negative(float x) {
    union {
        unsigned int ui;
        float d;
    } my_union = { .d = x };
    return my_union.ui & 0x80000000;
}
```

非类型安全往往还和未定义行为（undefined behavior）联系在一起，常常出现在没有显式错误处理的语言中。对应的典型漏洞利用（security exploits）如 stack smashing attack ([stack buffer overflow]) 和 format string attack ([uncontrolled format string])。

同样需要注意的是，设计上的类型安全不等于真正的类型安全，主要出于以下原因：

- 对于绝大多数语言，很难做到完整的形式化证明（参考 [Featherweight Java]）
- 语言在实现上可能存在 bug
- 链接库可能使用其他语言

现在，基本清楚这三个概念在说些什么了，我们就来看看一些常见的编程语言分别属于哪一类。

| Programming Language | Strong/Weak Type | Static/Dynamic Typing | Type Safety/Type Unsafety |
| -------------------- | ---------------- | --------------------- | ------------------------- |
| C                    | Strong (?)       | Static                | Unsafety                  |
| Java                 | Strong (?)       | Static                | Safety                    |
| Python               | Strong (?)       | Dynamic               | Safety                    |
| JavaScript           | Weak             | Dynamic               | Unsafety                  |
| Haskell              | Strong           | Static                | Safety                    |

? : 对于哪些隐性类型转换是可接受的存在争议

[stack buffer overflow]: https://en.wikipedia.org/wiki/Stack_buffer_overflow
[uncontrolled format string]: https://en.wikipedia.org/wiki/Uncontrolled_format_string
[Featherweight Java]: https://dl.acm.org/doi/10.1145/503502.503505

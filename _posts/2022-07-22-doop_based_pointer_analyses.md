---
layout: post
title:  "基于 Doop 的指针分析与不同的上下文敏感性"
date:   2022-07-22
categories: jekyll update
---

## Doop based pointer analyses

- [Doop based pointer analyses](#doop-based-pointer-analyses)
  - [Doop](#doop)
    - [Datalog 与指针分析](#datalog-与指针分析)
    - [指针分析规范](#指针分析规范)
    - [优化与评估](#优化与评估)
  - [Bean](#bean)
  - [Scaler](#scaler)
  - [其他](#其他)

今天和大家分享的是指针分析的三个工作，包括 Doop 框架和基于它的两个工作。在 OOPSLA 2009 的文章 [Strictly Declarative Specification of Sophisticated Points-to Analyses](https://dl.acm.org/doi/10.1145/1640089.1640108) 中，Yannis Smaragdakis 等人提出了 Doop 这个指针分析框架，截止到 2022 年 7 月这篇文章有 386 的引用量，也是今天主要介绍的工作。另两个工作是南京大学的李樾、谭添老师在博士和博后阶段的工作，分别是和 [Jingling Xue](https://www.cse.unsw.edu.au/~jingling/) 合作的发表在 SAS 2016 的 [Making k-Object-Sensitive Pointer Analysis More Precise with Still k-Limiting](https://link.springer.com/chapter/10.1007/978-3-662-53413-7_24) (Bean)，和 Anders Møller 与 Yannis Smaragdakis 合作的发表在 FSE 2018 的 [Scalability-First Pointer Analysis with Self-Tuning Context-Sensitivity](https://dl.acm.org/doi/10.1145/3236024.3236041) (Scaler)。[李樾](https://yuelee.bitbucket.io/)、[谭添](https://silverbullettt.bitbucket.io/)和 [Jingling Xue](https://www.cse.unsw.edu.au/~jingling/) 一直都在做指针分析，还有一系列的文章（基础指针分析的会议文章包括 PLDI17，OOPSLA18，OOPSLA19，OOPSLA21，ASE21，ECOOP22 等，还有期刊文章以及基于指针分析的应用）。今天简要介绍的是较早的两篇，主要看一下，不同于 Doop 这种比较基础的工作，一个比较小的基础（非应用）创新点是如何考虑的。

### Doop

首先介绍 OOPSLA 2009 Doop 的工作。

#### Datalog 与指针分析

Doop 是一个基于 Datalog 的指针分析。

Datalog 是一个演绎数据库和逻辑编程语言，在数据库层面可以看作支持完整递归的 SQL，即增删改基于规则和事实两部分，例如事实 `parent(bob, alice)`，`parent(canon, bob)` 和规则 `ancestor(A, B) :- parent(A, B)` 可以推出 `ancestor(bob, alice)`，`ancestor(canon, bob)`，当然 parent 事实本身也在数据库里。我们增加规则 `ancestor(A, B) :- parent(A, C), ancestor(C, B)`，可以额外推出 `ancestor(canon, alice)`。事实是关系（relation），即谓词（predicate），例如 `parent(bob, alice)` 对应命题逻辑“bob 是 alice 的父辈”。规则由规则头、蕴含符号和规则体组成，规则头是关系原子（relational atom），规则体则包含一个或多个子目标（关系原子或算数原子（如 a > 0）），例如规则 `ancestor(A, B) :- parent(A, B)` 对应一阶逻辑：任意 A 是 B 的父辈，则 A 是 B 的祖先。Datalog 根据输入的事实推导出所有的关系。在语言层面，Datalog 可以看作没有函数的 Prolog。Datalog 也没有表达高阶逻辑的能力（本质是一阶谓词逻辑中 Horn 子句逻辑的一种受限形式），只允许变量或常量作为谓词的自变元，不允许函数作为谓词的自变元。（对应到集合，命题（零阶）逻辑只有对象（命题），一阶逻辑增加性质，即对象的集合，二阶逻辑则可以表示性质的集合。）

在本文中，关系用大写开头，变量用 ? 开头，蕴含写作 <-。Doop 的输入是 Java 程序（bytecode 形式，部分流敏感利用 Soot Jimple SSA IR），输出 Datalog 关系。例如，`new` 语句会生成 `AssignHeapAllocation(?var, ?heap)`，赋值语句会生成 `Assign(?from, ?to)`。这种映射由语法得到，在此基础上指针分析可以表示为例如规则：

```
VarPointsTo(?var, ?heap) <- AssignHeapAllocation(?var, ?heap).
VarPointsTo(?to, ?heap) <- Assign(?from, ?to), VarPointsTo(?from, ?heap).
```

这种表示是递归和声明式的，简洁清晰，同时可以表达例如可达性关系和指向关系这种互相依赖的分析（互递归定义）。Datalog 程序的求值可以对应关系代数的 join 和 projection，是一个两层循环，join 循环可以应用相对成熟的数据库优化技术。本文的 Datalog 引擎是商业版，[Doop](https://github.com/KnowSciEng/doop) 也提供开源版。商业版（有条件地）支持否定词和函数。

#### 指针分析规范

Datalog 可以自然地描述形式化的指针分析逻辑规范。有些规范是优化相关的，例如 Java 类的静态初始化方法的上下文无关处理不会影响指向分析的上下文敏感性，但是可以减少运行时间，这种策略会影响逻辑规范，生成的 Datalog 程序是不等价的。

首先考虑前面的规则的扩展：

```
VarPointsTo(?ctx, ?var, ?heap) <-
    AssignHeapAllocation(?var, ?heap, ?inmethod),
    CallGraphEdge(_, _, ?ctx, ?inmethod).
VarPointsTo(?toCtx, ?to, ?heap) <- 
    Assign(?fromCtx, ?from, ?toCtx, ?to, ?type),
    VarPointsTo(?fromCtx, ?from, ?heap),
    HeapAllocation:Type[?heap] = ?heaptype,
    AssignCompatible(?type, ?heaptype).
```

这里主要包含以下几个扩展：
- 上下文敏感：每个变量之前都带有其上下文。
- 在线的调用图构造（和指针分析是 Datalog 互递归），此前的工作一般先独立地在 Java 代码中得到调用图，但是没有指针分析的调用图是不精确的，基于不精确的调用图的指向分析也不够精确。
- 函数作为关系，函数 `Type[?heap] = ?heaptype` 等价于关系 `Type(?heap, ?heaptype)`，结合 : 可以表示一组谓词。
- 赋值考虑类型系统，即变量不能指向其类型不支持的对象抽象。

Doop 的一个特点是支持完整的 Java 语义，且不依赖 Datalog 以外的分析。Doop 建模了 Paddle（[Evaluating the Benifits of Context-Sensitive Points-to Analysis Using a BDD-based Implementation](https://dl.acm.org/doi/10.1145/1391984.1391987)）基于 BDD 的指向分析中的语言特性。BDD（二元决策图）是一种可以压缩指向关系集合的数据结构，我们不在这里介绍。对于 Paddle 已有的特性（如终止机制 finalization、特权操作 PrivilegedAction、线程等），Doop 取得更好的精度，同时支持更多特性，如弱引用和引用队列、反射、异常分析（另有一篇 ISSTA09），底层代码的方法调用等。Datalog 声明式方法的好处还在于增加语义扩展不会影响已有的分析，同时分析规范和语言规范也更贴近，例如以下是 Doop 中类型转换检查（类型 A 能否转换为类型 B）的部分代码。

```
/** If S is an ordinary (nonarray) class, then:
 *     o If T is a class type, then S must be the
 *       same class as T, or a subclass of T.
 */
CheckCast(?s, ?s) <- ClassType(?s).
CheckCast(?s, ?t) <- Subclass(?t, ?s).
/**    o If T is an interface type, then S must
 *       implement interface T.
 */
CheckCast(?s, ?t) <- ClassType(?s),
Superinterface(?t, ?s).
...
```

#### 优化与评估

声明式的规范把分析逻辑和算法实现分开，优化问题可以表示为等价的（即逻辑规范固定，部分规范考虑了优化）Datalog 程序的程序变换，即 join 的顺序会对性能产生影响。单规则的 join 顺序调整是自动的，规则间的优化需要 Datalog 引擎支持一些非标准的特性，例如支持引入新的数据库索引，同时依赖人工，所以这里我们不做深入。

对比当时 state-of-the-art 的 Java 指向分析框架 Paddle，Doop 取得了更好的精度、速度和完整性。Doop 集成了上下文敏感度可选的不同分析，在使用和 Paddle 同精度（1-call-site sensitive）的条件下，Doop 比其快 15 倍。在采用相关工作中最好的精度时，Doop 也比 Paddle 快 10 倍。Paddle 使用的 BDD 是一种可规约的数据结构，相比于此前用稀疏位向量存储指向集合的方法，BDD 主要是用相对小的时间增加得到了较大的内存减少，但是本文认为声明式的显式表示在时间和空间上都优于 BDD。

### Bean

SAS 2016，Making k-Object-Sensitive Pointer Analysis More Precise with Still k-Limiting。

一般认为，对于指针分析，流敏感对 C/C++ 程序提升更大，上下文敏感对 Java 等面向对象语言提升更大。上下文敏感又有不同的抽象表示和敏感度。

**k-call-site sensitive** 抽象的是调用栈，k 限制栈的深度，记录的是位置和上下文。以下面的程序为例：

```
n1 = new One(); // o1
n2 = new Two(); // o2
x1 = newX(n1); // 3
x2 = newX(n2); // 4
n = x1.f;

X newX(Number P) {
    X x = new X(); // o8
    x.f = p;
    return x;
}
class X {
    Number f;
}
```

当上下文不敏感时，p 和 n 都是不精确的：

| Variable | Object |
| -------- | ------ |
| n1       | o1     |
| n2       | o2     |
| p        | o1, o2 |
| x        | o8     |
| x1       | o8     |
| x2       | o8     |
| n        | o1, o2 |

当 1-call-site sensitive 时，p 区分调用点变得精确，但是 n 仍然是不精确的：

| Variable | Object |
| -------- | ------ |
| n1       | o1     |
| n2       | o2     |
| 3:p      | o1     |
| 3:x      | o8     |
| x1       | o8     |
| 4:p      | o2     |
| 4:x      | o8     |
| x2       | o8     |
| o8.f     | o1, o2 |
| n        | o1, o2 |

这就引出了 **context-sensitive heap**，即不仅变量有上下文，（堆）对象也有上下文，这可以使得 n 也得到精确的结果：

| Variable | Object |
| -------- | ------ |
| n1       | o1     |
| n2       | o2     |
| 3:p      | o1     |
| 3:x      | 3:o8   |
| x1       | 3:o8   |
| 4:p      | o2     |
| 4:x      | 4:o8   |
| x2       | 4:o8   |
| 3:o8.f   | o1     |
| 4:o8.f   | o2     |
| n        | o1     |

变量上下文仅考虑调用位置也有不能解决的问题，比如：

```
a1 = new A(); // o1
a2 = new A(); // o2
b1 = new B(); // o3
b2 = new B(); // o4
a1.set(b1); // 5
a2.set(b2); // 6
x = a1.get();

class A {
    B f;
    void set(B b) {
        doSet(b);
    }
    void doSet(B p) {
        this.f = p; // 12
    }
    B get() {
        return this.f;
    }
}
```

当 1-call-site sensitive 时，x 不能得到精确的结果：

| Variable      | Object |
| --------      | ------ |
| 5:set_this    | o1     |
| 5:b           | o3     |
| 6:set_this    | o2     |
| 6:b           | o4     |
| 12:doSet_this | o1, o2 |
| 12:p          | o3, o4 |
| o1.f          | o3, o4 |
| o2.f          | o3, o4 |
| x             | o3, o4 |

**k-object sensitive** 记录调用对象（而非调用位置）和上下文，从而得到 x 的精确结果：

| Variable       | Object |
| --------       | ------ |
| o1:set_this    | o1     |
| o1:b           | o3     |
| o2:set_this    | o2     |
| o2:b           | o4     |
| o1:doSet_this  | o1     |
| o1:p           | o3     |
| o2:doSet_this  | o2     |
| o2:p           | o4     |
| o1.f           | o3     |
| o2.f           | o4     |
| x              | o3     |

对于指针分析的精度，k-object sensitive 并不理论一定优于 k-call-site sensitive，但一般在面向对象语言中有更好的结果。

k-call-site sensitive 和 k-object sensitive 的变量上下文敏感都可以和 context-sensitive heap 的堆上下文敏感结合在一起。对于 Java 的指针分析，一般认为 2obj+h 是精度和规模性权衡下最好的。

Doop 包含不同上下文敏感度的指针分析变体，例如前面没有给出的 `CallGraphEdge` 的规则：

```
CallGraphEdge(?callerCtx, ?call, ?calleeCtx, ?callee) <-
    VirtualMethodCall:Base[?call] = ?base,
    VirtualMethodCall:SimpleName[?call] = ?name,
    VirtualMethodCall:Descriptor[?call] = ?descriptor,
    VarPointsTo(?callerCtx, ?base, ?heap),
    HeapAllocation:Type[?heap] = ?heaptype,
    MethodLookup[?name, ?descriptor, ?heaptype] = ?callee,
    ?calleeCtx = ?call.
```

`?calleeCtx = ?call` 可知这是一个 1-call-site sensitive 的例子。

回到这篇文章，针对 2obj 的一个不足，对于程序：

```
void main(Object[] args) {
    A a1 = new A(); // A/1
    Object v1 = a1.foo(new Object()); // O/1

    a2 = new A(); // A/2
    Object v2 = a2.foo(new Object()); // O/2
}
class A {
    Object foo(Object v) { // 9
        B b = new B(); // B/1
        return b.bar(v);
    }
}
class B {
    Object bar(Object v) { // 15
        C c = new C(); // C/1
        return c.identity(v);
    }
}
class C {
    Object identity(Object v) { return v; } // 21
}
```

2-object sensitive 得到的 v1 和 v2 是不精确的：

| Variable     | Object   |
| --------     | ------   |
| a1           | A/1      |
| a2           | A/2      |
| [A/1]v9      | O/1      |
| [A/2]v9      | O/2      |
| [A/1,B/1]v15 | O/1      |
| [A/2,B/1]v15 | O/1      |
| [B/1,C/1]v21 | O/1, O/2 |
| v1           | O/1, O/2 |
| v2           | O/1, O/2 |

这种不精确在 3-object sensitive 下可以消除：

| Variable         | Object   |
| --------         | ------   |
| a1               | A/1      |
| a2               | A/2      |
| [A/1]v9          | O/1      |
| [A/2]v9          | O/2      |
| [A/1,B/1]v15     | O/1      |
| [A/2,B/1]v15     | O/2      |
| [A/1,B/1,C/1]v21 | O/1      |
| [A/2,B/1,C/1]v21 | O/2      |
| v1               | O/1      |
| v2               | O/2      |

很明显 2obj 丢失了第 3 层的调用信息，但是 3obj 在多数情况下可能并不必要，同时我们可以观察到在 `A/1,B/1,C/1` 和 `A/2,B/1,C/1` 中，`B/1` 是冗余的，它会被 2obj 保留下来却不能区分不同的调用上下文。

本文设计了一种图结构 OAG（Object Allocation Graph），对于上例：

```
       A/1
      /   \
O_root     B/1 -- C/1
      \   /
       A/2
```

通过图上的算法去掉冗余的节点 `B/1`，使得 2obj 得到 `A/1,C/1` 和 `A/2,C/1`。

虽然实现是在 Doop 中做的，但是本文没有沿用 Doop 的形式规范描述，其形式化章节很值得学习，并给出了 Bean-2obj 在最坏情况下不弱于 2obj 的证明。

### Scaler

FSE 2018，Scalability-First Pointer Analysis with Self-Tuning Context-Sensitivity。

除了 call-site sensitive 和 object sensitive，还有 **type sensitive** 记录的是调用对象实例所属的类和上下文。对于这么多不同上下文敏感性的 Java 的指针分析，本文发现在一般情况下有：精度 object > type > call-site > insensitive，效率 insensitive > type > object > call-site。而且 2type 和 2obj 都存在分析超时的情况，即规模性较差。

本文的思路就是对每个方法使用不同的敏感性。通过一个上下文不敏感的指向分析即可得到 OAG，从而预估指向边和上下文的乘积，按照预置的上下文敏感变体排列（精度上升，规模性下降），如 {1type, 2type, 2obj}，根据用户给定的规模性度量阈值（保存指向关系的内存占用），选择精度最高的上下文敏感性。

### 其他

Java 有几乎是通用的基准测试集 DaCapo，其他相对新的语言（如 JavaScript、Python、Go）有吗？相对于爬很多库，基准测试集除了可以大量减少实验评估中的非技术工作量，也可以方便不同工作的横向比较和复现。
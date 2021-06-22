---
layout: post
title:  "静态分析中头文件的处理"
date:   2020-07-14
categories: jekyll update
---

为了使源码可编译，首先需要预处理器（preprocessor）处理 `#include`、`#define` 等命令（directive）并移除注释。但是常常会有一些头文件找不到或 `typedef` 未知的奇怪问题。

### 一个预处理的简单例子

例如，我们要编译 [python-ldap](https://github.com/python-ldap/python-ldap) 的 C 扩展模块。作为 Python 项目整体打包时，我们只需要执行 `python setup.py install`，而从 `setup.py` 知道扩展模块包含的源码和头文件之后，直接调用 C 编译器单独预处理这部分 C 代码却不成功：

以 gcc 为例（clang 和 cpp 命令类似）

`gcc -E <file.c>`

首先一个问题就是 `'Python.h' file not found`，这是 Python 外部函数接口 [Python/C API](https://docs.python.org/3/c-api/index.html#c-api-index) 定义的入口，而 C 编译器并不知道 Python 头文件的位置。所以我们用 `-I <dir>` 参数指定一个 include 搜索路径：

`gcc -E -I<3.6/include/python3.6m> file.c`

具体路径可以用 `find / -name 'Python.h'` 搜索，同理补全更多的搜索路径之后就可以通过编译器的预处理阶段了。

### 解析预处理的输出

预处理不是目的，一般还需要转换到某些中间形式才能进行静态分析，以使用 [pycparser](https://github.com/eliben/pycparser)（或 [libclang](https://clang.llvm.org/doxygen/group__CINDEX.html)）生成 AST 为例，在解析（parse）上一步得到的预处理后的代码时，往往又会产生新的报错，例如系统头文件 `<MacOSX.sdk/usr/include/i386/_types.h>` 或 `<MacOSX.sdk/usr/include/lber.h>` 中一些 `typedef` 未知，报错形如 `before: __attribute__` 或 `before: __OSX_AVAILABLE_BUT_DEPRECATED_MSG`。这里有两个问题，其一是这些头文件并不在我们的搜索路径中，这是因为 gcc 有一些预置的系统头文件目录，使用编译参数 `-nostdinc` 可以禁用这种行为。其二是既然预处理器没有报错，为什么其输出无法解析，这是因为 pycparser 不支持 GCC 等编译器扩展，比如 GNU-specific `__attribute__`，为此我们可以使用 `-D` 参数补充这块定义，比如 `-D'__attribute__(x)='`。

pycparser 使用 `fake_libc_include` 处理一些标准 C 库头文件，其中给出一些 `#define` 和 `typedef` 的最小化定义：

`typedef int T;`

即解析器（parser）只需要知道存在类型 `T`，而不在乎该类型的具体实现（语义），比如 `T` 可能是一个接受结构体数组的函数指针。

pycparser 支持完整的 C99 语言，如果目标程序只依赖**标准 C 库头文件**（standard C library headers，C runtime，如 `stdio.h`），那么 `-I<fake_libc_include>` 一般就够了。但是如前例所示，目标程序可能还依赖**系统头文件**（system headers，如 `_types.h`）和**其他库的头文件**（other library headers，如 `Python.h`），这些就需要通过编译参数补充，这些补充定义也可以是最小化的。一般来说，对于我们关心的部分（假设是目标程序和 Python/C API）应该提供完整定义，而其他部分就可以使用最小化定义，这样既可以省去大量繁琐工作，也可以减少解析时间和 AST 体积。

### 一个复杂的例子

我们尝试预处理并解析 Python 源码中 `Modules` 目录下 C 语言实现的标准库，按照以上策略补充了一系列参数之后依旧存在两个问题，其一是一些定义和平台产生了冲突（如 [definition wrong for platform](https://github.com/python/cpython/blob/master/Include/pyport.h#L743)）。其二是补充定义的量过大而无法穷尽，一般是存在大量系统相关的定义（如 [__declspec](https://docs.microsoft.com/en-us/cpp/cpp/declspec?view=vs-2019)、[\_\_asm\_\_](https://stackoverflow.com/questions/26456510/what-does-asm-volatile-do-in-c)），这里其实已经变成了一个交叉编译（cross compilation）的问题（TODO）。

一个简单的替代解决方法是先在本地机器上执行构建（build）Python 的配置脚本：

`./configure --with-pydebug`

即产生平台相关的 `pyconfig.h` 来解决上述问题，之后重复以上策略配置少量参数就可以得到目标代码的 AST 了。

### 一个小工具

一方面，有一些通用的编译参数，也有一些目标项目特定的，对于同一项目希望可以实现参数的复用，同时不直接修改而是复用 `fake_libc_include`；另一方面，希望配置的过程可以遵循一定的流程，避免输入非常长的命令，并且不和分析部分代码耦合。因此，基于 pycparser 和 yaml 实现了一个编译参数配置工具，可以自动解析配置文件中的编译参数，动态构建 `pycparser.parse_file()` 的参数列表。

主程序 `preprocess` 除目标文件路径外还接受一个可选参数。缺省时仅加载配置文件 `pp_config.yml` 中 `all` 对象存储的通用参数，指定 `<project_name>` 时则会加载项目特定参数。调试过程就是反复执行主程序，根据报错信息增量地修改配置文件中对应对象的参数条目。调试成功后输出 C 目标程序的 AST，编译参数也自动在配置文件中保留了下来。

### 参考

1. [pycparser](https://github.com/eliben/pycparser)
2. [On parsing C, type declarations and fake headers](https://eli.thegreenplace.net/2015/on-parsing-c-type-declarations-and-fake-headers)
3. [How to preprocess a C source code for pycparser](https://stackoverflow.com/questions/33763611/how-to-preprocess-a-c-source-code-for-pycparser)
4. [用 \_\_attribute\_\_((deprecated)) 管理过时的代码](https://blog.csdn.net/benkaoya/article/details/52368638)
5. [\_\_attribute\_\_ 总结](https://www.jianshu.com/p/29eb7b5c8b2d)
6. [Include/pyport.h: Bad LONG_BIT assumption on non-glibc sys](https://bugs.python.org/issue1023838)
7. [OSXCross: Fix building GCC with 10.15 SDK, __OSX_AVAILABLE_BUT_DEPRECATED_MSG](https://github.com/tpoechtrager/osxcross/commit/f4b0948abd9cad576d2def3d1cd52b9ef956ef52)
8. [Microsoft-specific modifiers: __declspec](https://docs.microsoft.com/en-us/cpp/cpp/declspec?view=vs-2019)
9. [What does \_\_asm\_\_ \_\_volatile\_\_ do in C](https://stackoverflow.com/questions/26456510/what-does-asm-volatile-do-in-c)
10. Build Python from source: [1](https://devguide.python.org), [2](https://devguide.python.org/setup/#compiling), [3](https://devguide.python.org/setup/#build-dependencies)
11. [yaml](http://www.ruanyifeng.com/blog/2016/07/yaml.html)

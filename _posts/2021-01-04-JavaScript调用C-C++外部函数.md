---
layout: post
title:  "JavaScript 调用 C/C++ 外部函数"
date:   2021-01-14
categories: jekyll update
---

本文主要包括以下内容：

- JS 调用 C/C++ 的方法和原理
- JS 外部函数调用的类型错误隐患
- JS/C 内存管理：单向透明的内存模型

JS 调用 C/C++ 需要通过 WebAssembly，假设有 C++ 文件 `function.cpp` 如下：

```c++
#include <math.h>

extern "C" {

int int_sqrt(int x) {
    return sqrt(x);
}

}
```

由于 C/C++ 函数名映射到 JS 端时默认是在前面加一个下划线，所以使用 `extern "C"` 避免 C++ 的名称重整（name mangling）。

使用 [emscripten](https://github.com/emscripten-core/emscripten) 编译（[文档](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#interacting-with-code)）：

```bash
emcc function.cpp -o function.html -s EXPORTED_FUNCTIONS='["_int_sqrt"]' -s EXPORTED_RUNTIME_METHODS='["ccall","cwrap"]'
```

`EXPORTED_FUNCTIONS` 指明需要暴露的对象，`EXPORTED_RUNTIME_METHODS` 指明除直接调用外，还想要使用运行时函数 `ccall` 和 `cwrap`。

编译后生成了一个 `.js` 文件，一个 `.wasm` 文件和一个 `.html` 文件。`function.js` 中有这样的胶水代码：

```javascript
function createExportWrapper(name, fixedasm) {
  return function() {
    var displayName = name;
    var asm = fixedasm;
    if (!fixedasm) {
      asm = Module['asm'];
    }
    assert(runtimeInitialized, 'native function `' + displayName + '` called before runtime initialization');
    assert(!runtimeExited, 'native function `' + displayName + '` called after runtime exit (use NO_EXIT_RUNTIME to keep it alive after main() exits)');
    if (!asm[name]) {
      assert(asm[name], 'exported native function `' + displayName + '` not found');
    }
    return asm[name].apply(null, arguments);
  };
}

/** @type {function(...*):?} */
var _int_sqrt = Module["_int_sqrt"] = createExportWrapper("int_sqrt");
};
```

其中 `Module` 是 Emscripten 运行时的全局对象，根据名字 `int_sqrt` 去由 C++ 编译得到的 `function.wasm` 中找对应的对象并执行：

```wat
(func (;1;) (type 0) (param i32) (result i32)
    (local i32 i32 i32 i32 i32 i32 i32 i32 i32 i32 i32 i32 f64 f64 f64)
    global.get 0
    local.set 1
    i32.const 16
    local.set 2
    local.get 1
    local.get 2
    i32.sub
    local.set 3
    local.get 3
    global.set 0
    local.get 3
    local.get 0
    i32.store offset=12
    local.get 3
    i32.load offset=12
    local.set 4
    local.get 4
    call 2
    local.set 13
    local.get 13
    f64.abs
    local.set 14
    f64.const 0x1p+31 (;=2.14748e+09;)
    local.set 15
    local.get 14
    local.get 15
    f64.lt
    local.set 5
    local.get 5
    i32.eqz
    local.set 6
    block  ;; label = @1
      block  ;; label = @2
        local.get 6
        br_if 0 (;@2;)
        local.get 13
        i32.trunc_f64_s
        local.set 7
        local.get 7
        local.set 8
        br 1 (;@1;)
      end
      i32.const -2147483648
      local.set 9
      local.get 9
      local.set 8
    end
    local.get 8
    local.set 10
    i32.const 16
    local.set 11
    local.get 3
    local.get 11
    i32.add
    local.set 12
    local.get 12
    global.set 0
    local.get 10
    return)

(export "int_sqrt" (func 1))
```

所以在 JS 端直接调用就是 `_int_sqrt(10)` 或 `Module._int_sqrt(10)`，这种方式更快，但是需要自己保证传入参数的类型和 wasm 实现相匹配（如此例中的 `i32`）。也可以使用 `ccall` 调用：

```javascript
var result = Module.ccall('int_sqrt', // C function name
                          'number',   // return type
                          ['number'], // argument types
                          [10]        // arguments
);
```

`ccall` 实现如下：

```javascript
// C calling interface.
/** @param {string|null=} returnType
    @param {Array=} argTypes
    @param {Arguments|Array=} args
    @param {Object=} opts */
function ccall(ident, returnType, argTypes, args, opts) {
  // For fast lookup of conversion functions
  var toC = {
    'string': function(str) {
      var ret = 0;
      if (str !== null && str !== undefined && str !== 0) { // null string
        // at most 4 bytes per UTF-8 code point, +1 for the trailing '\0'
        var len = (str.length << 2) + 1;
        ret = stackAlloc(len);
        stringToUTF8(str, ret, len);
      }
      return ret;
    },
    'array': function(arr) {
      var ret = stackAlloc(arr.length);
      writeArrayToMemory(arr, ret);
      return ret;
    }
  };

  function convertReturnValue(ret) {
    if (returnType === 'string') return UTF8ToString(ret);
    if (returnType === 'boolean') return Boolean(ret);
    return ret;
  }

  var func = getCFunc(ident);
  var cArgs = [];
  var stack = 0;
  assert(returnType !== 'array', 'Return type should not be "array".');
  if (args) {
    for (var i = 0; i < args.length; i++) {
      var converter = toC[argTypes[i]];
      if (converter) {
        if (stack === 0) stack = stackSave();
        cArgs[i] = converter(args[i]);
      } else {
        cArgs[i] = args[i];
      }
    }
  }
  var ret = func.apply(null, cArgs);

  ret = convertReturnValue(ret);
  if (stack !== 0) stackRestore(stack);
  return ret;
}

function writeArrayToMemory(array, buffer) {
  assert(array.length >= 0, 'writeArrayToMemory array must have a length (should be an array or typed array)')
  HEAP8.set(array, buffer);
}
```

给出了入参为 `string` 和 `array` 类型时分配 JS 堆的方式，外部函数 `stackAlloc` 计算存储位置：

```javascript
/** @type {function(...*):?} */
var stackAlloc = Module["stackAlloc"] = createExportWrapper("stackAlloc");
```

其 wasm 实现如下：

```wat
(func (;5;) (type 0) (param i32) (result i32)
  (local i32 i32)
  global.get 0
  local.get 0
  i32.sub
  i32.const -16
  i32.and
  local.tee 1
  global.set 0
  local.get 1)

(export "stackAlloc" (func 5))
```

这被称作[单向透明的内存模型](https://www.cntofu.com/book/150/zh/ch2-c-js/ch2-03-mem-model.md)，即 C/C++ 编译得到的 wasm 的运行时堆和运行时栈全部在 JS 堆（`Module.buffer`）上，JS 环境中的其他对象无法被 wasm 直接访问。

`cwrap` 是对 `ccall` 的封装，方便多次调用：

```javascript
int_sqrt = Module.cwrap('int_sqrt', 'number', ['number']);
int_sqrt(10);
```

`cwrap` 实现如下：

```javascript
/** @param {string=} returnType
    @param {Array=} argTypes
    @param {Object=} opts */
function cwrap(ident, returnType, argTypes, opts) {
  return function() {
    return ccall(ident, returnType, argTypes, arguments, opts);
  }
}
```

此外我们可以看到不管是直接调用还是通过 `ccall` 调用，对于参数个数都没有检查，同时对于非 `string` 和 `array` 类型的参数也没有类型检查（`string` 和 `array` 类型的检查机制需要进一步考察内部的函数调用）。

此例接收一个整型参数，我们构造如下的错误：

```javascript
var result = Module.ccall('int_sqrt',
                          'number',
                          ['number'],
                          ['10']
);
console.log(result); // 1

int_sqrt = Module.cwrap('int_sqrt', 'number', ['number']);
console.log(int_sqrt(10, 5)); // 2

console.log(_int_sqrt('a')); // 3
```

程序点 1 输出 3（10 的平方根取整），这与 JS 的弱类型是一致的；程序点 2 输出 3 是因为没有参数个数检查，直接忽略了第二位置开始的所有参数；程序点 3 输出 0。三者都没有报错。

我们进一步探究一下内存模型。

设计 C++ 程序如下：

```c++
#ifndef EM_PORT_API
#	if defined(__EMSCRIPTEN__)
#		include <emscripten.h>
#		if defined(__cplusplus)
#			define EM_PORT_API(rettype) extern "C" rettype EMSCRIPTEN_KEEPALIVE
#		else
#			define EM_PORT_API(rettype) rettype EMSCRIPTEN_KEEPALIVE
#		endif
#	else
#		if defined(__cplusplus)
#			define EM_PORT_API(rettype) extern "C" rettype
#		else
#			define EM_PORT_API(rettype) rettype
#		endif
#	endif
#endif

#include <stdio.h>

int g_int = 42;
double g_double = 3.1415926;

EM_PORT_API(int*) get_int_ptr() {
	return &g_int;
}

EM_PORT_API(double*) get_double_ptr() {
	return &g_double;
}

EM_PORT_API(void) print_data() {
	printf("C{g_int:%d}\n", g_int);
	printf("C{g_double:%lf}\n", g_double);
}
```

JS 调用和操作如下：

```javascript
Module.onRuntimeInitialized = () => {
  console.log("=== orginal C value ===");
  Module._print_data();

  console.log("=== read C memory from JS side ===");

  var int_ptr = Module._get_int_ptr();
  var int_value = Module.HEAP32[int_ptr >> 2];
  console.log("JS{int_value:" + int_value + "}");

  var double_ptr = Module._get_double_ptr();
  var double_value = Module.HEAPF64[double_ptr >> 3];
  console.log("JS{double_value:" + double_value + "}");

  console.log("=== write C memory from JS side ===");
  Module.HEAP32[int_ptr >> 2] = 13;
  Module.HEAPF64[double_ptr >> 3] = 123456.789;
  Module._print_data();
};
```

输出为：

```
=== orginal C value ===
C{g_int:42}
C{g_double:3.141593}
=== read C memory from JS side ===
JS{int_value:42}
JS{double_value:3.1415926}
=== write C memory from JS side ===
C{g_int:13}
C{g_double:123456.789000}
```

可以看到，在 JS 中正确读取了 C/C++（wasm）的内存数据；JS 中写入的数据，在 wasm 中亦能正确获取。

需要注意的是，`Module.buffer` 是一个 `ArrayBuffer` 对象，是保存二进制数据的一维数组，无法直接访问，必须通过某种类型的 TypedArray 方可对其进行读写。可以理解为 `ArrayBuffer` 是实际存储数据的容器，在其上创建的 TypedArray 则是把该容器当作某种类型的数组来使用。常用对应关系如下表：

| Object | TypedArray | C datatype |
| ------ | ---------- | ---------- |
| Module.HEAP8 | Int8Array | int8 |
| Module.HEAP32 | Int32Array | int32 |
| Module.HEAPU16 | Uint16Array | uint16 |
| Module.HEAPF64 | Float64Array | double |

所以在上例中，在 JS 中通过各种类型的 HEAP 对象视图访问 wasm 的内存数据时，地址必须对齐。

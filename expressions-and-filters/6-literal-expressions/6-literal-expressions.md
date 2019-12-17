下面要开始实现我们的 Angular 表达式解释器，先实现一个版本，这个版本可以解析字面量表达式——也就是能自解释的数据表达式，就像数字，字符串，或者一个字面量数组（"42", [1, 2, 3]），而变量和函数调用则不是（`fourtyTwo`，`a.b()`）。

字面量表达式是我们能进行解析的、最简单的表达式。通过先支持这种表达式的解析，我们可以把构成表达式解析功能的各部分结构和动态填充上去。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter5-scope-events)
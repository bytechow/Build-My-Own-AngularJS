下面开始实现我们的 Angular 表达式解析器。首先，我们实现一个可以解析字面量表达式（能“自解释”的数据表达式）的简单版本，比如数字、字符串或者一个字面量数组（"42", [1, 2, 3]），像变量和函数调用就不是字面量了（`fourtyTwo`，`a.b()`）。

字面量表达式是解析起来最简单的表达式。支持解析字面量表达式后，我们就可以把表达式解析功能的各个结构和动力组件填充上去。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter5-scope-events)
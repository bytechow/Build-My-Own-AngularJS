### 初始配置
#### Setup

解析 Angular 表达式的代码将会放到一个叫 `src/parse.js` 的新文件中，它的名字来源于它将提供的 `$parse` 服务。

在这个文件中，会有一个对外暴露（external-facing）的函数，被称为 `parse`。它接收一个 Angular 表达式字符串，然后返回一个能在指定上下文中执行表达式的函数：

_src/parse.js_

```js
'use strict';

function parse(expr) {
  // return ...
}

module.exports = parse;
```

> 后面依赖注入实现并且能正常运行之后，我们会把这个函数变成 `$parse` 服务。

这个文件会包含四个对象，用于将表达式转换为函数：一个是 Lexer，一个是 AST Builder，一个是 AST Compiler，还有一个是 Parse。它们分别负责不同阶段的工作：

Lexer 会接收一个原始的表达式字符串，然后返回一个从这个表达式中解析出来的 token （符号）数组。举个例子，字符串 `“a + b”` 就会被解释为三个 token：`a`、`+` 和 `b`。

`AST Builder` 会接收由 Lexer 生成的 token 数组，并依此构建一棵 `Abstract Syntax Tree`（AST，抽象语法树）。这棵树使用嵌套的 JavaScript 对象来表示表达式的语法结构。比如，`a`、`+` 和 `b` 三个 token 就会被转换为类似下面的东西：

```js
{
  type: AST.BinaryExpression,
  operator: '+',
  left: {
    type: AST.Identifier,
    name: 'a'
  },
  right: {
    type: AST.Identifier,
    name: 'b'
  }
}
```

AST Compiler 会接收一棵抽象语法树，然后把它编译为一个 JavaScript 函数，这个函数可以语法树形式的表达式进行运算。例如，上面的 AST 可能会被转化成下面的东西：

```js
function(scope) {
  return scope.a + scope.b;
}
```

`Parser` 的职责就是把上面几个基础步骤给串起来。它本身没有干什么事情，“重活”都交给 Lexer、AST Builder 和 AST Compiler 做了。
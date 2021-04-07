### 初始配置
#### Setup

我们会把解析 Angular 表达式的代码放到 `src/parse.js` 文件中，文件的名字来源于它对外提供的 `$parse` 服务。

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

![expression-parsing](/assets/6-literal-expressions/expression-parsing.png)

这意味着无论你在 Angular 应用中的何处使用了表达式，背后都代表着一个 JavaScript 函数被生成了。在之后的 digest 中，一旦需要对表达式进行求值，这些函数就会被再次执行。

下面我们会为上述这些功能准备一些“脚手架”。首先，`Lexer` 会被定义为一个构造函数。它包含了一个叫 `lex` 的方法，这个方法会执行 tokenization （表达式符号拆分）的过程：

_src/parse.js_

```js
function Lexer() {
}

Lexer.prototype.lex = function(text) {
  // Tokenization will be done here
};
```

而 AST Builder（在代码中使用的名称就是 `AST`）则是另一个构造函数。他会接收一个 Lexer 作为参数。它也有一个名为 `ast` 的方法，这个方法会根据表达式转化而出的 token 执行 AST 的构建：

_src/parse.js_

```js
function AST(lexer) {
  this.lexer = lexer;
}

AST.prototype.ast = function(text) {
  this.tokens = this.lexer.lex(text);
  // AST building will be done here
};
```
### 初始配置
#### Setup

我们会新建一个文件 `src/parse.js` 来存放解析 Angular 表达式的代码，文件的名字来源于它对外提供的 `$parse` 服务。

在文件中，我们只会对外暴露（external-facing）一个 `parse` 函数。它接收一个 Angular 表达式，然后返回一个能在指定上下文中执行表达式的函数：

_src/parse.js_

```js
'use strict';

function parse(expr) {
  // return ...
}

module.exports = parse;
```

> 后面一旦依赖注入器能启动并正常运行时，我们会把这个函数转换成 `$parse` 服务。

这个文件包含了四个用于将表达式转换为函数的对象：一个是 Lexer（词法分析器），一个是 AST Builder（AST 构建器），一个是 AST Compiler（AST 编译器），还有一个是 Parser（语法解析器）。它们分别负责不同阶段的工作：

_Lexer_ 会接收一个原始的表达式字符串，然后返回一个数组，数组元素是从这个表达式中解析出来的 token（符号）。举个例子，字符串 `“a + b”` 会被解析为三个 token：`a`、`+` 和 `b`。

`AST Builder` 会接收由 Lexer 生成的 token 数组，并依此构建一棵 `Abstract Syntax Tree`（AST，抽象语法树）。这棵树使用嵌套的 JavaScript 对象来表示表达式的语法结构。比如，`a`、`+` 和 `b` 三个 token 就会被转换为类似下面这样的东西：

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

_AST Compiler_ 会接收一棵抽象语法树，然后把它编译为一个 JavaScript 函数，这个函数会对这个
语法树对应的表达式进行运算。上面的 AST 可能会被转化成：

```js
function(scope) {
  return scope.a + scope.b;
}
```

`Parser` 的职责就是把上面几个基础步骤给串起来而已。它本身没什么工作量，“重活”都委托给 Lexer、AST Builder 和 AST Compiler 去做了。

![expression-parsing](/assets/6-literal-expressions/expression-parsing.png)

这意味着，每当我们在 Angular 中使用了表达式，背后都会有一个 JavaScript 函数生成。在之后的 digest 期间一旦需要对表达式进行求值，这些函数就会被重复执行。

下面我们先来准备一个“脚手架”。首先，`Lexer` 会被定义为一个构造函数。它包含了一个名为 `lex` 的方法，这个方法会执行 tokenization（表达式符号拆分）的过程：

_src/parse.js_

```js
function Lexer() {
}

Lexer.prototype.lex = function(text) {
  // Tokenization will be done here
};
```

AST Builder（在代码中就叫 `AST`）是另一个构造函数。他会接收一个 Lexer 作为参数。它也有一个同名方法（名为 `ast`），这个方法会根据表达式拆分出来的 token 执行 AST 的构建：

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

AST Compiler 也是一个构造函数，它会接收一个 AST Builder 实例作为参数。AST Compiler 有一个名为 `compile` 的方法，用于把表达式编译为表达式函数：

_src/parse.js_

```js
function ASTCompiler(astBuilder) {
  this.astBuilder = astBuilder;
}

ASTCompiler.prototype.compile = function(text) {
  var ast = this.astBuilder.ast(text);
  // AST compilation will be done here
};
```

`Parser` 其实也是一个构造函数，它会串起上述流程，形成一条完整的解析管道。它会接收一个 Lexer 实例作为参数，同时也有一个同名方法——`parse`：

_src/parse.js_

```js
function Parser(lexer) {
  this.lexer = lexer;
  this.ast = new AST(this.lexer); this.astCompiler = new ASTCompiler(this.ast);
}

Parser.prototype.parse = function(text) {
  return this.astCompiler.compile(text);
};
```

现在我们终于可以写公共的 `parse` 函数了，它会创建一个 Lexer 和 Parser，然后调用 `Parser.parse` 方法：

_src/parse.js_

```js
function parse(expr) {
  var lexer = new Lexer();
  var parser = new Parser(lexer);
  return parser.parse(expr);
}
```

这就是整个 `parse.js` 文件的顶层结构。在本章剩余部分和接下来几章中，我们会往这个函数里填充一些有趣的特性，最终实现魔术般的效果。
### 解析整数
#### Parsing Integers

整数字面量是最简单的字面量，比如 `42`。正因为整数表达式最简单，因此我们把它作为实现解析器的第一步。

老规矩，先新增一个测试用例来描述我们想实现的效果。创建 `test/parse_spec.js` 文件，然后把下面内容加入进去：

_test/parse_spec.js_

```js
'use strict';

var parse = require('../src/parse');

describe('parse', function() {

  it('can parse an integer', function() {
    var fn = parse('42');
    expect(fn).toBeDefined();
    expect(fn()).toBe(42);
  });

});
```

我们在文件开头启用 ECMAScript 严格模式，然后从 `parse.js` 中导入 `parse` 函数。在单元测试中，我们对 `parse` 的行为进行了定义。这个函数会对原始的字符串进行运算，得到一个经过解析后的结果值。

要实现这个功能，我们需要先考虑 Lexer 的输出是什么。之前我们已经说过了 Lexer 会输出一个 token 集合，但 token 究竟是什么？

对我们的目标而言，token 就是一个对象，它为 AST Builder 提供构建抽象语法树所需的所有信息。要解析整数字面量，我们只需要两样东西：

- 需要进行 token 解析的源文本
- token 的数字类型值

对数字 42 来说，token 可以被简单解析为：

```js
{
  test: '42',
  value: 42
}
```

因此，我们希望 Lexer 对数字进行解析后能输出一个跟上面类似的数据结构。

Lexer 的 `lex` 函数的内容基本上就是一个大循环语句，它会遍历字符串中的每一个字符。遍历的同时，它会把字符串转化成一个 token 集合：

_src/parse.js_

```js
Lexer.prototype.lex = function(text) {
  this.text = text;
  this.index = 0;
  this.ch = undefined;
  this.tokens = [];

  while (this.index < this.text.length) {
    this.ch = this.text.charAt(this.index);
  }

  return this.tokens;
};
```

这个函数目前还什么都没干（除了会无限循环以外），但它准备了在循环中会用到的几个局部变量：

- `text` - 原始的字符串
- `index` - 当前遍历到的字符索引
- `ch` - 当前遍历到的字符
- `tokens` - token 的结果集

我们会在 `while` 循环中处理各种各样的字符。先来处理一下数字：

_src/parse.js_

```js
Lexer.prototype.lex = function(text) {
  // this.text = text;
  // this.index = 0;
  // this.ch = undefined;
  // this.tokens = [];

  while (this.index < this.text.length) {
    // this.ch = this.text.charAt(this.index);
    if (this.isNumber(this.ch)) {
      this.readNumber();
    } else {
      throw 'Unexpected next character: ' + this.ch;
    }
  }

  // return this.tokens;
};
```

如果当前字符是一个数字，我们会使用一个叫 `readNumber` 的函数来处理它。如果这个字符不是数字，由于目前我们还无法解决，所以直接抛出异常就好了。

检查函数 `isNumber` 也十分简单：

_src/parse.js_

```js
Lexer.prototype.isNumber = function(ch) {
  return '0' <= ch && ch <= '9';
};
```

我们使用数字运算符 `<=` 来检查这个字符的值是否处于 `‘0’` 到 `‘9’` 之间。与数字比较不同，JavaScript 会在此处使用字典序比较，但在只有个位数的情况下，它们两者是相同的。

`readNumber` 方法的顶层结构跟 `lex` 很类似：它会对文本进行逐个字符的检索，并在遍历过程中构建数字：

_src/parse.js_

```js
Lexer.prototype.readNumber = function() {
  var number = '';
  while (this.index < this.text.length) {
    var ch = this.text.charAt(this.index);
    if (this.isNumber(ch)) {
      number += ch;
    } else {
      break;
    }
    this.index++;
  }
};
````

这个 `while` 循环会读取当前字符。如果这个字符是一个数字，它会把这个数字加入到本地变量 `number` 中去，然后将字符索引自增 1。如果字符不是数字，就终止这个循环。

这能生成一个 `number` 字符串，但目前我们还不会对它进行处理。我们需要对外输出一个 token：

_src/parse.js_

```js
Lexer.prototype.readNumber = function() {
  // var number = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
  //   if (this.isNumber(ch)) {
  //     number += ch;
  //   } else {
  //     break;
  //   }
  //   this.index++;
  // }
  this.tokens.push({
    text: number,
    value: Number(number)
  });
};
```

这里我们把一个新的 token 添加到 `this.tokens` 这个集合中去。这个 token 的 `text` 属性就是我们之前读取过的字符串，而 `value` 属性就是把这个字符串通过 `Number 构造函数` 转化而成的数字值。

现在，Lexer 已经完成了在解析数字过程中属于自己的任务。下面我们来看看 AST 构建器。

正如之前介绍的，AST 是一个嵌套的 JavaScript 对象，它使用类似树的形式来表示表达式。这棵树上的每一个节点都会有一个 `type` 属性来描述这个节点代表的语法结构。除了类型信息以外，节点上还会有该类型特定的一些属性，这些属性能提供更多关于这个节点的信息。

举个例子，数字字面量的类型是 `AST.Literal`，而它的 `value` 属性就保存了这个字面量的值：

```js
{type: AST.Literal, value: 42}
```

每一个 AST 都有一个类型为 `AST.Program` 的根节点。根节点上有一个 `body` 属性用于存放表达式的内容。因此，实际上这个数字字面量要被包裹到 `AST.Program` 节点下面去：

```js
{
  type: AST.Program,
  body: {
    type: AST.Literal,
    value: 42 
  }
}
```

这就是目前我们需要应该从 Lexer 输出中生成的 AST。

> 实际上，我们现在还跳过了 AST 的另一层包裹。它跟含有多个 `statements`（声明）的表达式有关。我们将在本书后面的部分看到它是如何操作的。

我们通过 AST 构建器的 `program` 方法来生成顶层的 Program 节点。它会作为整个 AST 构建过程的返回值：

_src/parse.js_

```js
AST.prototype.ast = function(text) {
  // this.tokens = this.lexer.lex(text);
  return this.program();
};
AST.prototype.program = function() {
  return {type: AST.Program};
};
```

AST.Program 类型的值，是一个在 `AST` 函数中定义的“标记常量”。它用于识别它所表示的节点类型。它的值其实就是一个简单的字符串而已：

_src/parse.js_

```js
function AST(lexer) {
  this.lexer = lexer;
}
AST.Program = 'Program';
```

之后我们会对所有 AST 节点类型引入类似的标记常量，然后在 AST 编译器中根据它们来判断应该生成哪种类型的 JavaScript 代码。

程序应该有程序体，在这种情况下程序体就是一个数字字面量而已。它的 `type` 就是 `AST.Literal`，而且它会由 AST 构建器的 `constant` 方法来生成：

_src/parse.js_

```js
AST.prototype.program = function() {
  return {type: AST.Program, body: this.constant()};
};
AST.prototype.constant = function() {
  return {type: AST.Literal, value: this.tokens[0].value};
};
```

目前我们只需要访问第一个 token，并获取它的 `value` 属性。

我们还需要添加这种节点类型的标记常量：

_src/parse.js_

```js
AST.Program = 'Program';
AST.Literal = 'Literal';
```

这样我们就得到了一个代表数字字面量的 AST 了，下面我们把注意力集中在 AST 编译器以及把这个 AST 转换为 JavaScript 函数上。

AST 编译器的任务是遍历 AST 构建器生成的树，并根据树节点生成对应的 JavaScript 源代码。然后它会根据源代码来生成一个 JavaScript 函数。对于数字字面量来说，它生成的函数将会非常简单：

```js
function() {
  return 42;
}
```

在 AST 编译器中主要用到的 `compile` 函数，我们会引入一个 `state` 属性来保存我们在遍历中收集到的信息。目前，我们只收集一样东西，也就是最终生成函数的函数体内容：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  this.state = {body: []};
};
```

一旦我们初始化好了 state，我们就可以开始对树进行遍历了，通过一个叫 `recurse` 的函数：

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = {body: []};
  this.recurse(ast);
};
ASTCompiler.prototype.recurse = function(ast) {

};
```

我们的目标是一旦 `recurse` 的调用结束了，`state.body` 会包含 JavaScript 语句，我们可以根据这些语句创建函数。而这个函数会成为我们的返回值：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  var ast = this.astBuilder.ast(text);
  this.state = {body: []};
  this.recurse(ast);
  /* jshint -W054 */
  return new Function(this.state.body.join(''));
  /* jshint +W054 */
};
```

我们使用 [Function 构造函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function)来创建这个函数。这个构造函数接收 JavaScript 源代码，然后把它们动态编译为一个函数。这基本上就是 `eval` 的一种方式。JSHint 并不接受 `eval`，所以我们需要显式地告诉它，我们已经知道这种做法有危险，不需要再报警告了。（`W054` 是 JSHint 中的一个数字类型的错误代码，它代表“Function 构造函数也是 eval 的一种方式”）。

最后我们需要再 `recurse` 函数中进行处理。我们希望在这里可以生成指定的 JavaScript 代码，并把这些代码放到 `this.state.body` 中去。

顾名思义，`recurse` 方法会对树节点进行递归遍历。由于每个节点都有一个 `type`，不同类型的节点需要不同的处理，我们会使用一个 `switch` 语法来切换对不同节点类型的逻辑处理分支：

_src/parse.js_

```js
ASTCompiler.prototype.recurse = function(ast) {
  switch (ast.type) {
    case AST.Program:
    case AST.Literal:
  }
};
```

字面量是一个“叶子节点”，也就是只包含了值而没有子节点的节点。所以我们可以直接返回这个节点的值：

_src/parse.js_

```js
case AST.Literal:
  return ast.value
```

而对于 Program 节点来说，我们要进行一些额外处理。我们需要为表达式生成一个 return 语句。我们需要返回的是 Program 的 `body` 属性值，我们可以通过 `recurse` 来得到这这个属性值：

_src/parse.js_

```js
case AST.Program:
  this.state.body.push('return ', this.recurse(ast.body), ';');
  break;
```

第一个单元测试的 body 恰好就是标志符 42 的值，因此最终返回的函数体为 `return 42`。

这样就可以通过这个单元测试了。我们已经可以生成字符串表达式 `‘42’` 的表达式函数了。

> 你可以通过调用表达式函数的 `toString()` 方法来验证由 `parse` 方法生成的源代码。这在调试更复杂的表达式时非常有用。

这里面有相当多的工作，可能有人会觉得没必要这么麻烦，但随着我们往表达式添加更多的特性，不同部分所承担的角色会越来越清晰。

> 你可能已经注意到了，我们这里只考虑了正整数。我们对负数会有不一样的处理，要考虑把负号视为一个运算符而不是数字的一部分。
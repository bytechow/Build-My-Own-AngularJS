### 解析整数

#### Parsing Integers

整数字面量是最简单的字面量，比如 `42`。正因为整数表达式最简单，我们把它作为实现解析器的第一步。

老规矩，先写一个测试用例来描述我们想实现的效果。创建 `test/parse_spec.js` 文件，然后把下面内容加入进去：

_test/parse\_spec.js_

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

在文件开头，我们会启用 ECMAScript 严格模式，然后从 `parse.js` 中导入 `parse` 函数。这个单元测试会对 `parse` 函数的行为进行定义。`parse.js` 函数会接收一个原始字符串，返回一个函数。执行这个函数就能得到原始字符串的解析结果。

要实现这个功能，我们要先想想 Lexer 输出的是什么。前面我们已经介绍过 Lexer 会输出一个 token 集合，但 token 究竟是什么呢？

对我们的目标而言，token 就是一个对象，它为 AST Builder 提供构建抽象语法树所需的所有信息。要解析整数字面量，我们只需要两样东西：

* 需要进行 token 解析的源文本
* token 的数字值

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

* `text` - 原始的字符串
* `index` - 当前遍历到的字符索引
* `ch` - 当前遍历到的字符
* `tokens` - token 的结果集

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

如果当前字符是一个数字，我们会委托一个叫 `readNumber` 的函数来处理它。目前我们还无法处理数字以外的字符，所以遇到之后直接抛出异常就好了。

`isNumber` 这个检查函数也很简单：

_src/parse.js_

```js
Lexer.prototype.isNumber = function(ch) {
  return '0' <= ch && ch <= '9';
};
```

我们使用数字运算符 `<=` 来检查这个字符值（在字典中的排序）是否处于 `‘0’` 到 `‘9’` 之间。JavaScript 在这里使用的字典序比较，而不是数字比较，但在个位数的情况下，它们的结果是相同的。

`readNumber` 方法的顶层结构跟 `lex` 很类似：它会逐个遍历文本中的字符，并在此过程中构建数字：

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
```

这个 `while` 循环会读取当前字符。如果这个字符是一个数字，它会把这个数字加入到本地变量 `number` 中去，字符索引向前进 1 位，否则就终止循环。

这样就生成了一个 `number` 字符串，目前我们还不会用到它。我们需要对外输出一个 token：

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

我们仅仅在 `this.tokens` 这个集合加入一个新的 token 而已。这个 token 的 `text` 属性是当前已完成读取的字符串，而 `value` 属性则是使用 [Number 构造函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number) 从该字符串转换过来的数值。

Lexer 已经为解析数字出一份力了，下面我们来看看 AST 构建器。

正如之前介绍的，AST 是一个嵌套的 JavaScript 对象结构，它使用类似树的形式来表示表达式。树中的每个节点都会有一个 `type` 属性，用于描述节点代表的句法结构。除了类型以外，节点上还会有跟当前类型相关的其他特定属性，这些属性提供了更多关于这个节点的信息。

举个例子，数字字面量节点的类型是 `AST.Literal`，而它的 `value` 属性就保存了这个字面量的值：

```js
{type: AST.Literal, value: 42}
```

每一个 AST 都有一个类型为 `AST.Program` 的根节点。这个根节点上有一个 `body` 属性，用于存放表达式的内容。因此，上面这个数字字面量节点实际上要被包裹在 `AST.Program` 节点之中：

```js
{
  type: AST.Program,
  body: {
    type: AST.Literal,
    value: 42 
  }
}
```

这就是我们需要应该从 Lexer 输出中生成的 AST。

> 实际上，我们现在还少包裹了一层，这个包裹层与有多个 `statements`（声明）的表达式有关，后面会讲到。

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

AST.Program 类型的值是一个在 `AST` 函数对象中定义的“标记常量“。这类型的常量就是用来标记节点的类型而已，本质上就是一个简单的字符串而已：

_src/parse.js_

```js
function AST(lexer) {
  this.lexer = lexer;
}
AST.Program = 'Program';
```

后面我们会为所有 AST 节点类型都定义一个这样的标记常量，然后在 AST 编译器中根据这个类型常量来判断应该生成哪种类型的 JavaScript 代码。

每个程序都会有自己的程序体，当前表达式的程序体就是一个数字字面量节点，这个节点的 `type` 是 `AST.Literal`，而它的代码会由 AST 构建器的 `constant` 方法来生成：

_src/parse.js_

```js
AST.prototype.program = function() {
  return {type: AST.Program, body: this.constant()};
};
AST.prototype.constant = function() {
  return {type: AST.Literal, value: this.tokens[0].value};
};
```

目前，我们只需要获取第一个 token 的 `value` 属性值就可以了。

我们还需要添加与这种节点类型对应的标记常量：

_src/parse.js_

```js
AST.Program = 'Program';
AST.Literal = 'Literal';
```

这样我们就得到了一个表示单个数字字面量的 AST 了。下面我们来看看 AST 编译器是如何把这个 AST 转换为 JavaScript 函数的。

AST 编译器要做的是遍历这棵 AST 构建器生成的树，然后根据树节点生成对应的 JavaScript 源代码，最后会根据源代码来生成一个 JavaScript 函数。一个数字字面量表达式生成的函数是非常简单的：

```js
function() {
  return 42;
}
```

在编译器主体函数——`compile` 函数中，我们会引入一个 `state` 属性来保存我们在遍历中收集到的信息。目前，我们只需要收集最终会构成函数体的 JavaScript 代码就可以了：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  this.state = {body: []};
};
```

初始化 state 属性之后，我们就可以开始对树进行遍历了，遍历要用到一个叫 `recurse` 方法：

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = {body: []};
  this.recurse(ast);
};
ASTCompiler.prototype.recurse = function(ast) {

};
```

一旦 `recurse` 调用结束，我们希望 `state.body` 已经包含了用于生成表达式函数的 JavaScript 语句，生成的函数将成为 `compile` 方法的返回值：

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

我们使用 [Function 构造函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function) 来创建这个函数。`Function` 构造函数接收 JavaScript 源代码，然后把它们动态编译为一个函数。这也算是 `eval` 的一种方式。JSHint 不允许使用 `eval`，所以我们需要明确地告诉它我们已经知道这种做法有危险，不需要再报警告了。（`W054` 是 JSHint 中的一种错误的数字代码，它表示“Function 构造函数也是 eval 的一种方式”）。

最后我们还要搞清楚 `recurse` 函数应该做些什么。`recurse` 方法要生成指定的 JavaScript 代码，并把这些代码存放在 `this.state.body` 中。

顾名思义，`recurse` 方法会对树节点进行递归遍历。由于每个节点都有一个 `type` 属性，不同类型的节点需要不同的处理方式，我们会借用 `switch` 语法来实现这个需求：

_src/parse.js_

```js
ASTCompiler.prototype.recurse = function(ast) {
  switch (ast.type) {
    case AST.Program:
    case AST.Literal:
  }
};
```

由于字面量是一个“叶子节点”（只有值而没有子节点的节点），我们可以直接返回这个节点的值：

_src/parse.js_

```js
case AST.Literal:
  return ast.value
```

我们还需要 Program 节点进行一些额外处理，我们需要为整个表达式生成一个 `return` 语句。返回值就是 Program 的 `body` 属性值，我们可以通过调用 `recurse` 得到这个属性值：

_src/parse.js_

```js
case AST.Program:
  this.state.body.push('return ', this.recurse(ast.body), ';');
  break;
```

第一个单元测试的 body 就是 `42`，因此最终返回的函数体为 `return 42`。

这样就可以通过单元测试了。现在我们已经成功地从字符串表达式 `‘42’` 生成了一个表达式函数了。

> 你可以通过对解析返回的表达式函数调用 `fn.toString()` 来检查实际生成的源代码。这在调试复杂表达式时非常有用。

这里面有相当多的模块一起协作，可能有人觉得没必要这么麻烦，但随着我们往里面添加更多的特性，不同部分所承担的角色将会越来越清晰。

> 你可能已经注意到了，我们在这里只考虑了正整数。我们需要对负数进行特殊处理，要把负号看作一个运算符而不是数字的一部分。

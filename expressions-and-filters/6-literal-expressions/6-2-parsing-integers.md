### 解析整数
#### Parsing Integers

最简单的字面量就是整数了，比如 `42`。实现整数表达式很简单，因此我们用它来作为实现解析器的第一步。

我们先来添加一个测试用例来描述我们想实现的效果。创建 `test/parse_spec.js` 文件，然后把下面内容加入进去：

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

我们在文件开头启用了 ECMAScript 严格模式，然后从 `parse.js` 中导入 `parse` 函数。在单元测试中，我们对 `parse` 的行为进行了定义。这个函数会对原始的字符串进行运算，得到一个经过解析后的结果值。

要实现这个功能，我们需要先考虑 Lexer 的输出是什么。之前我们已经说过了 Lexer 会输出一个 token 列表，但 token 究竟是什么？

对（要实现 Angular 表达式的）我们来说，token 就是一个能提供构建抽象语法树所需信息的对象。目前，要实现对这个整数字面量进行解析，我们只需要两样东西：

- 我们需要对其进行 token 转换的文本
- token 的数字类型值

对数字 42 来说，token 可以被简化为：

```js
{
  test: '42',
  value: 42
}
```

所以，如果我们要在 Lexer 中对数字值进行解析，就需要设法让它输出一个跟上面类型的数据结构。

Lexer 的 `lex` 函数基本上是由一个体量较大的循环语句组成，它会对给定字符串的字符进行遍历。在遍历的过程中，它会不停地把字符串转化为一个 token 集合：

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

这个函数目前还什么都没干（除了会无限循环以外），但它为我们准备了会在循环中用到的几个局部变量：

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

现在，Lexer 已经完成了在解析数字过程中属于自己的任务。下面我们来看看 AST Builder。

正如之前介绍的，AST 是一个嵌套的 JavaScript 对象，它使用类似树的形式来表示表达式。这棵树上的每一个节点都会有一个 `type` 属性来描述这个节点代表的语法结构。除了类型信息以外，节点上还会有该类型特定的一些属性，这些属性能提供更多关于这个节点的信息。

举个例子，数字字面量的类型是 `AST.Literal`，而它的 `value` 属性就保存了这个字面量的值：

```js
{type: AST.Literal, value: 42}
```
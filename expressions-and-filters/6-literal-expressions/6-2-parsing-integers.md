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
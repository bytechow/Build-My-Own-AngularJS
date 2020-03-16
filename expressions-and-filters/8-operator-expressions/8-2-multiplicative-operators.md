### 多元运算符
#### Multiplicative Operators

在一元运算符以后，拥有更高优先级的操作符实多元运算符：乘法，除法和取余。不出所料，它们的工作原理都与 JavaScript 一样：

```js
it('parses a multiplication', function() { expect(parse('21 * 2')()).toBe(42);
});

it('parses a division', function() { expect(parse('84 / 2')()).toBe(42);
});

it('parses a remainder', function() { expect(parse('85 % 43')()).toBe(42);
});
```

首先我们会把这些操作符放到 `OPERATORS` 的集合中，这样它们才能够从 Lexer 中弹出：

_src/parse.js_

```js
var OPERATORS = {
  '+': true,
  '!': true,
  '-': true,
  '*': true,
  '/': true,
  '%': true
```

AST 构建器会使用一个新方法 `multiplicative` 来处理这些操作符。它会发出一个 `BinaryExpression` 节点（表示表达式由两个参数组成）。表达式的左右两边都将会是一个一元表达式：

_src/parse.js_

```js
AST.prototype.multiplicative = function() {
  var left = this.unary();
  var token;
  if ((token = this.expect('*', '/', '%'))) {
    left = {
      type: AST.BinaryExpression,
      left: left,
      operator: token.text,
      right: this.unary()
    };
  }
  return left;
};
```

像之前很多 AST 构建器的方法一样，这个方法也会有一个“回退”模式：如果表达式与多元表达式的格式不相匹配，就会返回一个一元表达式。

我们需要增加这个新的 AST 节点：

_src/parse.js_

```js
// AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression';
// AST.ObjectExpression = 'ObjectExpression';
// AST.Property = 'Property';
// AST.Identifier = 'Identifier';
// AST.ThisExpression = 'ThisExpression';
// AST.LocalsExpression = 'LocalsExpression';
// AST.MemberExpression = 'MemberExpression';
// AST.CallExpression = 'CallExpression';
// AST.AssignmentExpression = 'AssignmentExpression';
// AST.UnaryExpression = 'UnaryExpression';
AST.BinaryExpression = 'BinaryExpression';
```
### 乘法类运算符
#### Multiplicative Operators

在一元运算符以后，拥有更高优先级的是乘法类运算符：乘法，除法和取余。不出所料，它们的工作原理都与 JavaScript 一样：

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

像之前很多 AST 构建器的方法一样，这个方法也会有一个“回退”模式：如果表达式与乘法类表达式的格式不相匹配，就会返回一个一元表达式。

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

现在，在 `AST.assignment` 中，我们会调用 `multiplicative` 而不是之前的 `unary`：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.multiplicative();
  // if (this.expect('=')) {
    var right = this.multiplicative();
  //   return { type: AST.AssignmentExpression, left: left, right: right };
  // }
  // return left;
};
```

然后我们就可以在 AST 编译器中支持乘法类运算符了。它们跟一元运算符基本一样，只不过乘法类运算符的两边各有一个运算数而已。

_src/parse.js_

```js
case AST.BinaryExpression:
  return '(' + this.recurse(ast.left) + ')' +
    ast.operator +
    '(' + this.recurse(ast.right) + ')';
```

在本章开头，我们已经讨论过优先级规则的重要性了。现在我们来看看应该如果去定义这些规则了。优先级规则并不是由在某个地方定义的“优先顺序表”来决定的，而是隐藏在 AST 构建函数调用彼此的顺序。

比如现在，我们“排在最顶层的” AST 构建函数是 `assignment`。然后，`assignment` 会调用 `multiplicative`，`multiplicative` 继而调用 `unary`，`unary` 会调用 `primary`。每个方法首先都会调用这个链条中的下一个函数来构建“左侧的”运算数。这意味着，在链条中的最后一个方法拥有最高优先级。目前，我们的优先级顺序如下：

1. 基本表达式（Primary）
2. 一元运算符（Unary）
3. 乘法类运算符（Multiplicative）
4. 赋值表达式（Assignment）

随着我们加入越多越多的运算符，我们调用函数的顺序就相当于定义它们的优先顺序。

我们已经能成功解析乘法类运算符了，但如果我们连续使用了多个乘法类运算符会怎么样呢？

_test/parse_spec.js_

```js
it('parses several multiplicatives', function() {
  expect(parse('36 * 2 % 5')()).toBe(2);
});
```

这个测试还没能通过，因为 `multiplicative` 仅仅能够解析最多一个运算数，并返回这个运算符。如果有运算符超过一个，剩下的运算数会被忽略。

要解决这个问题，我们需要在发现有多个操作符需要处理时，就需要在 `multiplicative` 里消耗所有的操作符。每一步的结果都会成为下一步的左侧运算数，并在抽象语法树增加一层。在实际操作上，我们只需要对 `AST.multiplicative` 的 `while` 循环判断条件进行调整就可以了：

_src/parse.js_

```js
AST.prototype.multiplicative = function() {
  // var left = this.unary();
  // var token;
  while ((token = this.expect('*', '/', '%'))) {
  //   left = {
  //     type: AST.BinaryExpression,
  //     left: left,
  //     operator: token.text,
  //     right: this.unary()
  //   };
  // }
  // return left;
};
```

由于这三个运算符的优先级相同，它们会从左至右依次执行，我们的函数本来就是这样做的。
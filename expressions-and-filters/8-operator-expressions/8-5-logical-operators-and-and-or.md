### 逻辑并和逻辑非运算符
#### Logical Operators AND and OR

接下来，我们要实现的两个逻辑操作符是 `&&` 和 `||`。它们的功能正如我们所期待的那样：

_test/parse_spec.js_

```js
it('parses logical AND', function() {
  expect(parse('true && true')()).toBe(true);
  expect(parse('true && false')()).toBe(false);
});

it('parses logical OR', function() {
  expect(parse('true || true')()).toBe(true);
  expect(parse('true || false')()).toBe(true);
  expect(parse('false || false')()).toBe(false);
});
```

跟其他二元运算一样，我们可以连续使用多个逻辑运算符：

_test/parse_spec.js_

```js
it('parses logical AND', function() {
  expect(parse('true && true')()).toBe(true);
  expect(parse('true && false')()).toBe(false);
});

it('parses logical OR', function() {
  expect(parse('true || true')()).toBe(true);
  expect(parse('true || false')()).toBe(true);
  expect(parse('false || false')()).toBe(false);
});
```

逻辑运算符的有趣之处是它们拥有的“短路”特性。如果逻辑并表达式的值为假值（falsy），那就不会再执行右侧的表达式了，跟它们在 JavaScript 的表现一模一样：

_test/parse_spec.js_

```js
it('short-circuits AND', function() {
  var invoked;
  var scope = { fn: function() { invoked = true; } };

  parse('false && fn()')(scope);
  
  expect(invoked).toBeUndefined();
});
```

而对于逻辑或运算符，如果它左侧的表达式结果为真值（truthy），那右侧表达式就不会被执行：

_test/parse_spec.js_

```js
it('short-circuits OR', function() {
  var invoked;
  var scope = { fn: function() { invoked = true; } };

  parse('true || fn()')(scope);
  
  expect(invoked).toBeUndefined();
});
```

逻辑并的优先级高于逻辑或：

_test/parse_spec.js_

```js
it('parses AND with a higher precedence than OR', function() { expect(parse('false && true || true')()).toBe(true);
});
```

这里我们测试的表达式的运算结果将会是 `(false && true) || true` 而不是 `false && (true || true)`。

相等性运算符的优先级高于逻辑运算符：

_test/parse_spec.js_

```js
it('parses OR with a lower precedence than equality', function() { expect(parse('1 === 2 || 2 === 2')()).toBeTruthy();
});
```

现在我们知道，开发的这些操作符都是有一套固定套路的。在 `OPERATORS` 对象中我们会多增加两个实体：

_src/parse.js_

```js
var OPERATORS = {
  // '+': true,
  // '-': true,
  // '!': true,
  // '*': true,
  // '/': true,
  // '%': true,
  // '=': true,
  // '==': true,
  // '!=': true,
  // '===': true,
  // '!==': true,
  // '<': true,
  // '>': true,
  // '<=': true,
  // '>=': true,
  '&&': true,
  '||': true
};
```

在 AST 构建器中，我们会增加两个新函数来构建 `LogicalExpression` 运算符——一个是处理逻辑或（OR），而另一个是处理逻辑并（AND）的：

_src/parse.js_

```js
AST.prototype.logicalOR = function() {
  var left = this.logicalAND();
  var token;
  while ((token = this.expect('||'))) {
    left = {
      type: AST.LogicalExpression,
      left: left,
      operator: token.text,
      right: this.logicalAND()
    };
  }
  return left;
};
AST.prototype.logicalAND = function() {
  var left = this.equality();
  var token;
  while ((token = this.expect('&&'))) {
    left = {
      type: AST.LogicalExpression,
      left: left,
      operator: token.text,
      right: this.equality()
    };
  }
  return left;
};
```

这里有一个新的节点类型 `LogicalExpression`：

_src/parse.js_

```js
AST.Program = 'Program';
AST.Literal = 'Literal';
AST.ArrayExpression = 'ArrayExpression';
AST.ObjectExpression = 'ObjectExpression';
AST.Property = 'Property';
AST.Identifier = 'Identifier';
AST.ThisExpression = 'ThisExpression';
AST.LocalsExpression = 'LocalsExpression';
AST.MemberExpression = 'MemberExpression';
AST.CallExpression = 'CallExpression';
AST.AssignmentExpression = 'AssignmentExpression';
AST.UnaryExpression = 'UnaryExpression';
AST.BinaryExpression = 'BinaryExpression';
AST.LogicalExpression = 'LogicalExpression';
```

由于我们是按照优先级顺序从低往高的次序逐个调用的，对逻辑表达式的处理会紧跟在 `assignment` 之后执行：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.logicalOR();
  // if (this.expect('=')) {
    var right = this.logicalOR();
  //   return { type: AST.AssignmentExpression, left: left, right: right };
  // }
  // return left;
};
```

在 AST 编译器中，我们会新建一个 `AST.LogicalExpression` 分支，在这个分支里面，我们会先对左侧内容进行递归，并把递归结果保存起来作为整个表达式的结果：

_src/parse.js_

```js
case AST.LogicalExpression:
  intoId = this.nextId();
  this.state.body.push(this.assign(intoId, this.recurse(ast.left)));
  return intoId;
```

然后它会判断左侧内容的运算结果，如果运算符是 `&&` 且运算结果为真（truthy），或运算符是 `||` 且运算结果为假（falsy），才会去执行对右侧内容的运算。如果对右侧内容执行了运算，则右侧内容运算结果值会成为整个表达式的结果值：

_src/parse.js_

```js
case AST.LogicalExpression:
  // intoId = this.nextId();
  // this.state.body.push(this.assign(intoId, this.recurse(ast.left)));
  this.if_(ast.operator === '&&' ? intoId : this.not(intoId), this.assign(intoId, this.recurse(ast.right)));
  // return intoId;
```

这里我们只需要使用之前实现的 `if` 方法就可以同时处理 `&&` 和 `||` 了！虽然逻辑并和逻辑或运算符本质上是二元运算符，但由于它们会发生这种“短路”现象，我们才不把它们视作 `BinaryExpression` 进行处理。
### 关系和相等运算符
#### Relational And Equality Operators

在算术运算符之后，优先级较高的是各种用于比较的运算符。对数字来说，有四种关系运算符：

_test/parse_spec.js_

```js
it('parses relational operators', function() {
  expect(parse('1 < 2')()).toBe(true);
  expect(parse('1 > 2')()).toBe(false);
  expect(parse('1 <= 2')()).toBe(true);
  expect(parse('2 <= 2')()).toBe(true);
  expect(parse('1 >= 2')()).toBe(false);
  expect(parse('2 >= 2')()).toBe(true);
});
```

无论对于数字还是其他类型的值来说，都会有相等性校验以及非相等性校验。Angular 表达式同时支持 JavaScript 中的严格和松散两种模式的相等性比较：

_test/parse_spec.js_

```js
it('parses equality operators', function() {
  expect(parse('42 == 42')()).toBe(true);
  expect(parse('42 == "42"')()).toBe(true);
  expect(parse('42 != 42')()).toBe(false);
  expect(parse('42 === 42')()).toBe(true);
  expect(parse('42 === "42"')()).toBe(false);
  expect(parse('42 !== 42')()).toBe(false);
});
```

在这两个操作符家族中，关系运算符的优先级更高：

_test/parse_spec.js_

```js
it('parses relationals on a higher precedence than equality', function() {
  expect(parse('2 == "2" > 2 === "2"')()).toBe(false);
});
```

这个测试的执行顺序是这样的：

1. 2 == “2” > 2 === “2”
2. 2 == false === “2”
3. false === “2”
4. false

而不是：

1. 2 == “2” > 2 === “2”
2. true > false
3. 1 > 0
4. true

> [ECMAScript 规范](http://www.ecma-international.org/ecma-262/5.1/%23sec-9.3)，当 `true` 和 `false ` 出现在数字运算时，`true` 会被视作 `1`，而 `false` 会被视作 `0`，这像上面的第三步那样。

关系运算符和相等性运算符的优先级都低于加法类运算符：

_test/parse_spec.js_

```js
it('parses additives on a higher precedence than relationals', function() {
  expect(parse('2 + 3 < 6 - 2')()).toBe(false);
});
```

这个测试可以检验执行顺序会像下面这样：

1. 2+3<6- 2
2. 5 < 4
3. false

而不是：


1. 2+3<6- 2
2. 2 + true - 2
3. 2+1- 2
4. 1

这八个新运算符都会被加入到 `OPERATORS` 对象中去：

_src/parse.js_

```js
var OPERATORS = {
  '+': true,
  '-': true,
  '!': true,
  '*': true,
  '/': true,
  '%': true,
  '==': true,
  '!=': true,
  '===': true,
  '!==': true,
  '<': true,
  '>': true,
  '<=': true,
  '>=': true
};
```

在 AST 构建器中，我们需要引入两个新函数——一个用于处理相等运算符，另一个用于处理关系型运算符。我们不能使用同一个函数来同时处理这两类运算符，否则会破坏我们的优先级规则。这两个函数有相似的构成：

_src/parse.js_

```js
AST.prototype.equality = function() {
  var left = this.relational();
  var token;
  while ((token = this.expect('==', '!=', '===', '!=='))) {
    left = {
      type: AST.BinaryExpression,
      left: left,
      operator: token.text,
      right: this.relational()
    };
  }
  return left;
};
AST.prototype.relational = function() {
  var left = this.additive();
  var token;
  while ((token = this.expect('<', '>', '<=', '>='))) {
    left = {
      type: AST.BinaryExpression,
      left: left,
      operator: token.text,
      right: this.additive()
    };
  }
  return left;
};
```

现在，相等运算时除了赋值运算以外最低优先级的运算，因此在处理赋值运算的函数中，我们需要委托给 `equality` 函数：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.equality();
  if (this.expect('=')) {
    var right = this.equality();
    return { type: AST.AssignmentExpression, left: left, right: right };
  }
  return left;
};
```

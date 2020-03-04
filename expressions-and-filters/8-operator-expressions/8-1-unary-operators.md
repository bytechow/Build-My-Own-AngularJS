### 一元运算符
#### Unary Operators

我们将从优先级最高的的运算符开始，然后按照优先级递减的顺序向下进行。首先要介绍的是我们已经实现的 primary 表达式：每当有函数被调用或者有计算型或非计算型的属性访问时，它们都会被最先进行计算。而在 primary 表达式之后的是一元运算符。

一元运算符就是只有一个操作数的运算符：

- `-` 会把它的操作数取反，比如 `-42` 或者 `-a`
- `+` 实际上没做什么，但它可以用于强调或让代码更清晰，比如 `+42` 或者 `+a`
- 非运算符会对操作数的布尔值进行取反操作，比如 `!true` 或 `!a`

由于 `+` 运算符最简单，因为下面我们先从它开始讲起，它会原封不动地返回操作数：

_test/parse_spec.js_

```js
it('parses a unary +', function() {
  expect(parse('+42')()).toBe(42);
  expect(parse('+a')({ a: 42 })).toBe(42);
});
```

在 AST 构建器中，我们会加入一个名为 `unary` 的新方法来处理一元元算符，如果当前处理的字符不是一元运算符，它就会回退到使用 `primary` 进行处理：

_src/parse.js_

```js
AST.prototype.unary = function() {
  if (this.expect('+')) {

  } else {
  return this.primary();
  }
};
```

`unary` 方法实际上会构建一个 `UnaryExpression` 的 token，它唯一的参数将会是一个 primary 表达式：

_src/parse.js_

```js
AST.prototype.unary = function() {
  // if (this.expect('+')) {
    return {
      type: AST.UnaryExpression,
      operator: '+',
      argument: this.primary()
    };
  // } else {
  //   return this.primary();
  // }
};
```

`UnaryExpression` 是一个新的 AST 节点类型：

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
```
### 三元运算符
#### The Ternary Operator

本章要实现的最后一个运算符（以及倒数第二个操作符）是 C 语言风格的三元运算符，这个运算符让我们可以根据测试表达式的结果来选择返回两个可选值中的哪一个：

_test/parse_spec.js_

```js
it('parses the ternary expression', function() {
  expect(parse('a === 42 ? true : false')({ a: 42 })).toBe(true);
  expect(parse('a === 42 ? true : false')({ a: 43 })).toBe(false);
});
```

三元运算符的优先级刚好比逻辑或（OR）运算符低一级，因此，我们会先运算逻辑或表达式：

_test/parse_spec.js_

```js
it('parses OR with a higher precedence than ternary', function() {
  expect(parse('0 || 1 ? 0 || 2 : 0 || 3')()).toBe(2);
});
```

我们也可以使用嵌套的三元运算符，虽然这样做可能会让代码结构变得不清晰：

_test/parse_spec.js_

```js
it('parses nested ternaries', function() {
  expect(
    parse('a === 42 ? b === 42 ? "a and b" : "a" : c === 42 ? "c" : "none"')({
      a: 44,
      b: 43,
      c: 42
    })).toEqual('c');
});
```

跟本章看到的大多数运算符不一样，三元运算符并不会以一个运算符函数的形式出现在 `OPERATORS` 对象中。这是因为三元运算表达式中会有两个不同的运算符：问号`？`和冒号`:`，这样便于我们在 AST 构建阶段进行检测。

到目前为止，我们还没有完成用 Lexer 发出过 `?` 字符，那该怎么做呢？我们会对 `Lexer.lex` 方法中用于判断文本 token 的判断条件进行修改：

_src/parse.js_

```js
} else if (this.is('[],{}:.()?')) {
```

在 `AST` 中，我入会引入一个新函数 `ternary` 来构建这个运算符。它会对三个运算符和它们之间的内容进行消费，然后抛出一个 `ConditionalExpression` 节点：

_src/parse.js_

```js
AST.prototype.ternary = function() {
  var test = this.logicalOR();
  if (this.expect('?')) {
    var consequent = this.assignment();
    if (this.consume(':')) {
      var alternate = this.assignment();
      return {
        type: AST.ConditionalExpression,
        test: test,
        consequent: consequent,
        alternate: alternate
      };
    }
  }
  return test;
};
```

注意，由于我们会把处于三元运算符“中间”和“右侧”的表达式作为赋值（assignment）表达式进行消费，所以这两个表达式可以是任意表达式。同时也要注意，虽然这个方法能够能够会回退到 `logicalOR`，淡入检测到表达式包含运算符 `?` 符号，`:` 符号就必须要出现，而且没有可以回退的方法。这是因为我们使用了 `consume` 来进行消费，如果没有找到匹配的字符，就会抛出异常。

我们需要引入 `ConditionalExpression` 类型：

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
// AST.BinaryExpression = 'BinaryExpression';
// AST.LogicalExpression = 'LogicalExpression';
AST.ConditionalExpression = 'ConditionalExpression';
```
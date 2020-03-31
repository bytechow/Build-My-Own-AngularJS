### 语句
#### Statements

在总结本章之前，我们来看看在一个表达式执行多个任务的情况。

到目前为止，我们讨论的表达式都只发生在一个表达式中，最终也只会返回一个结果值。但这并不是一个严格的限制。实际上，在一个表达式字符串中我们可以有多个独立的表达式，只需要用分号分割它们即可：

_test/parse_spec.js_

```js
it('parses several statements', function() {
  var fn = parse('a = 1; b = 2; c = 3');
  var scope = {};
  fn(scope);
  expect(scope).toEqual({ a: 1, b: 2, c: 3 });
});
```

若我们这样写的话，最后一个表达式的结果值将会成为这个复合表达式的结果值。前边任何一个表达式的结果值都会被丢弃：

_test/parse_spec.js_

```js
it('returns the value of the last statement', function() {
  expect(parse('a = 1; b = 2; a + b')({})).toBe(3);
});
```

这意味着,如果有多个表达式，那除了最后一个，之前所有的表达式都可能产生副作用，比如属性赋值或函数调用。除了消耗 CPU 周期（CPU cycles）以外，没有产生任何实际影响，在命令式程序设计语言中，像这样的结构通常被称为语句，而不是表达式。要实现语句，我们首先要让 Lexer 可以抛出分号，这样我们才可以在 AST 构建器中识别这种符号。我们需要把这个文本 token 字符放到 `Lexer.lex` 这个不断扩展的集合中：

_src/parse.js_

```js
} else if (this.is('[],{}:.()?;')) {
```

在 AST 构建器中，我们需要改变 `AST.Program` 节点类型的性质，使其主体不再是单个表达式，而是一个表达式数组。只要能够匹配到它们之间的分号，我们就可以持续对表达式进行消耗，形成 body 数组：

_src/parse.js_

```js
AST.prototype.program = function() {
  var body = [];
  while (true) {
    if (this.tokens.length) {
      body.push(this.assignment());
    }
    if (!this.expect(';')) {
      return { type: AST.Program, body: body };
    }
  }
};
```

当表达式没有分号（这是最常见的情况），在循环结束时 `body` 数组应该只有一个元素。

在编译时，除了 body 中最后一条语句，每一个语句都以分号做结尾。然后，为 body 中的最后一个语句生成一个返回语句：

_src/parse.js_

```js
case AST.Program:
  _.forEach(_.initial(ast.body), _.bind(function(stmt) {
    this.state.body.push(this.recurse(stmt), ';');
  }, this));
  this.state.body.push('return ', this.recurse(_.last(ast.body)), ';');
  // break;
```

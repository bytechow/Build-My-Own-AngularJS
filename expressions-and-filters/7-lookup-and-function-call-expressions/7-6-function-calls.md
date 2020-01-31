### 函数调用
#### Function Calls

在 Angular 表达式中，函数调用与属性查找一样常用：

_test/parse_spec.js_

```js
it('parses a function call', function() {
  var fn = parse('aFunction()');
  expect(fn({aFunction: function() { return 42; }})).toBe(42);
});
```

首先要理解的事，函数调用实际上包含两个事件：

首先你要找到要调用的函数，像上面的 `‘aFunction’`，然后我们需要使用括号来进行调用。查找函数的过程跟其他属性查找并没有区别。毕竟在 JavaScript 中，函数跟其他值没有区别。

这就意味着我们可以利用现有的代码来完成对函数的查找，剩下来的工作就是调用。我们先把括号变成能被 Lexer 弹出的字符 token：

_src/parse.js_

```js
} else if (this.is('[],{}:.()')) {
  this.tokens.push({
    text: this.ch
  });
  this.index++;
```

跟访问属性一样，函数调用会作为一个 primary AST 节点进行处理。在 `AST.primary` 的 `while` 循环中，我们不仅要消耗方括号和点运算符，还要消耗圆括号。当遇到圆括号时，我们会生成一个 `CallExpression` 节点，然后把前一个 primary 表达式作为被调用函数（callee）：

_src/parse.js_

```js
// var next;
while ((next = this.expect('.', '[', '('))) {
  if (next.text === '[') {
    // primary = {
    //   type: AST.MemberExpression,
    //   object: primary,
    //   property: this.primary(),
    //   computed: true
    // };
    // this.consume(']');
  } else if (next.text === '.') {
    // primary = {
    //   type: AST.MemberExpression,
    //   object: primary,
    //   property: this.identifier(),
    //   computed: false
    // };
  } else if (next.text === '(') {
    primary = { type: AST.CallExpression, callee: primary };
    this.consume(')');
  }
}
```

我们还需要定义常量 `CallExpression`：

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
AST.CallExpression = 'CallExpression';
```
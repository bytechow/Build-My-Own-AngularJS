### 使用括号来调整优先级顺序
#### Altering The Precedence Order with Parentheses

当然，有时候我们可能不想要程序按照正常的优先级顺序执行，那 Angular 也提供了像 JavaScript 和其他很多语言类似的方法来改变优先级顺序，那就是通过括号来括起要优先执行的操作：

_test/parse_spec.js_

```js
it('parses parentheses altering precedence order', function() {
  expect(parse('21 * (3 - 1)')()).toBe(42);
  expect(parse('false && (true || true)')()).toBe(false);
  expect(parse('-((a % 2) === 0 ? 1 : 2)')({ a: 42 })).toBe(-1);
});
```

要实现它实际上非常简单。正因为括号会绕过整个优先级表格，我们在构建表达式和子表达式时，第一件事情就要检查一下是否有括号的存在。具体来说，我们需要在 `primary` 函数里检查是否存在括号。

如果在基本表达式（primary）里发现一个左括号，我们需要把括号里面的表达式都放到一个新的优先级链（precedence chain）中。这样我们就能保证括号里表达式会早于表达式中的其他内容进行运算：

_src/parse.js_

```js
AST.prototype.primary = function() {
  var primary;
  if (this.expect('(')) {
    primary = this.assignment();
    this.consume(')');
  } else if (this.expect('[')) {
  //   primary = this.arrayDeclaration();
  // } else if (this.expect('{')) {
  //   primary = this.object();
  // } else if (this.constants.hasOwnProperty(this.tokens[0].text)) {
  //   primary = this.constants[this.consume().text];
  // } else if (this.peek().identifier) {
  //   primary = this.identifier();
  // } else {
  //   primary = this.constant();
  // }
  // var next;
  // while ((next = this.expect('.', '[', '('))) {
  //   if (next.text === '[') {
  //     primary = {
  //       type: AST.MemberExpression,
  //       object: primary,
  //       property: this.primary(),
  //       computed: true
  //     };
  //     this.consume(']');
  //   } else if (next.text === '.') {
  //     primary = {
  //       type: AST.MemberExpression,
  //       object: primary,
  //       property: this.identifier(),
  //       computed: false
  //     };
  //   } else if (next.text === '(') {
  //     primary = {
  //       type: AST.CallExpression,
  //       callee: primary,
  //       arguments: this.parseArguments()
  //     };
  //     this.consume(')');
  //   }
  // }
  // return primary;
};
```
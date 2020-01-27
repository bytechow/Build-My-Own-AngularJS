### 非计算的属性查找

#### Non-Computed Attribute Lookup

除了引用作用域属性，我们还可以用点运算符查找嵌套数据结构中的、更深层的内容：

_test/parse\_spec.js_

```js
it('looks up a 2-part identifier path from the scope', function() {
  var fn = parse('aKey.anotherKey');
  expect(fn({ aKey: { anotherKey: 42 } })).toBe(42);
  expect(fn({ aKey: {} })).toBeUndefined();
  expect(fn({})).toBeUndefined();
});
```

我们希望这个表达式能够访问到 `aKey.anotherKey`，或者在一个甚至多个 key 缺失时返回 `undefined`。

一般来说，属性查找表达式不一定都是由标识符组成。它也可能是其他类型的表达式，比如对象字面量：

_test/parse\_spec.js_

```js
it('looks up a member from an object', function() {
  var fn = parse('{aKey: 42}.aKey');
  expect(fn()).toBe(42);
});
```

我们不会在 Lexer 中 emit 这个点运算符 token，所以在做其他事情之前，我们需要对此进行改变：

_src/parse.js_

```js
} else if (this.is('[],{}:.')) {
  this.tokens.push({
    text: this.ch
  });
  this.index++;
```

在 AST 构建时，这种非计算的属性查找表达式也被认为是一个 primary 节点，并在 `primary` 方法中进行处理。在处理完最初的 primary 节点后，我们需要检测它是否紧跟着一个点运算符：

_src/parse.js_

```js
AST.prototype.primary = function() {
  var primary;
  // if (this.expect('[')) {
    primary = this.arrayDeclaration();
  // } else if (this.expect('{')) {
    primary = this.object();
  // } else if (this.constants.hasOwnProperty(this.tokens[0].text)) {
    primary = this.constants[this.consume().text];
  // } else if (this.peek().identifier) {
    primary = this.identifier();
  // } else {
    primary = this.constant();
  // }
  if (this.expect('.')) {

  }
  return primary;
};
```

如果发现后面跟着一个点运算符，这个 primary 表达式会变成一个 `MemberExpression` 节点。我们会把这个 primary 节点作为成员表达式的对象，然后希望在点运算符后发现一个标识符，我们会把这个标识符当作属性名进行检索：

_src/parse.js_

```js
AST.prototype.primary = function() {
  // var primary;
  // if (this.expect('[')) {
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
  // if (this.expect('.')) {
    primary = {
      type: AST.MemberExpression,
      object: primary,
      property: this.identifier()
    };
  // }
  // return primary;
};
```

我们需要引入 `MemberExpression`：

_src/parse.js_

```js
// AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression';
// AST.ObjectExpression = 'ObjectExpression';
// AST.Property = 'Property';
// AST.Identifier = 'Identifier';
// AST.ThisExpression = 'ThisExpression';
AST.MemberExpression = 'MemberExpression';
```




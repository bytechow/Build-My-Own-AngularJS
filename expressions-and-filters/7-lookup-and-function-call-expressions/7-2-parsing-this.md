### 解析 this
#### Parsing this

还有一种特殊的属性检索方式需要我们进行特殊的处理，就是对 `this` 的引用。`this` 在 Angular 表达式中的角色与它在 JavaScript 中的角色类似：那就是运算表达式的上下文。表达式函数的上下文总是指向它运算时传入的作用域，因此 `this` 就是指向这个作用域：

_test/parse_spec.js_

```js
it('will parse this', function() {
  var fn = parse('this');
  var scope = {};
  expect(fn(scope)).toBe(scope);
  expect(fn()).toBeUndefined();
});
```

在 Lexer 中，`this` 要当作一个标识符进行处理，而在 AST 构建器中，我们现在需要在 `constants` 检索对象（lookup object）的加入一种特殊的 AST 节点：

_src/parse.js_

```js
AST.prototype.constants = {
  // 'null': { type: AST.Literal, value: null },
  // 'true': { type: AST.Literal, value: true },
  // 'false': { type: AST.Literal, value: false },
  'this': { type: AST.ThisExpression }
};
```

我们还没加入 `AST.ThisExpression`，所以咱们先加入：

_src/parse.js_

```js
AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression';
// AST.ObjectExpression = 'ObjectExpression';
// AST.Property = 'Property';
// AST.Identifier = 'Identifier';
```

在 AST 编译器的 `recurse` 方法中，我们可以把 `AST.ThisExpression` 节点编译为 `s` —— 也就是传入到表达式函数的作用域：

_src/parse.js_

```js
case AST.ThisExpression:
  return 's';
```
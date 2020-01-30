### 计算属性查找
#### Computed Attribute Lookup

上面我们已经看到如何使用点运算符对作用域上的属性进行访问，也就是以非计算属性查找的方式进行访问。我们还可以在 Angular 表达式中使用计算属性查找的方式，也就是方括号的方法来访问属性：

_test/parse_spec.js_

```js
it('parses a simple computed property access', function() {
  var fn = parse('aKey["anotherKey"]');
  expect(fn({ aKey: { anotherKey: 42 } })).toBe(42);
});
```

我们也可以用相同的方式来访问数组元素。你只需要把字符串换成数字就好了：

_test/parse_spec.js_

```js
it('parses a computed numeric array access', function() { 
  var fn = parse('anArray[1]');
  expect(fn({anArray: [1, 2, 3]})).toBe(2);
});
```

方括号表示法可能在解析时最有用，因为键本身会通过在作用域查找或其他方式计算出来（因此得名）。但你无法通过点运算符的方式来达到这种效果：

_test/parse_spec.js_

```js
it('parses a computed access with another key as property', function() { 
  var fn = parse('lock[key]');
  expect(fn({key: 'theKey', lock: {theKey: 42}})).toBe(42);
});
```

最后，这种表示法需要足够灵活，可以允许更复杂的表达式（比如其他计算属性访问）作为键：

_test/parse_spec.js_

```js
it('parses computed access with another access as property', function() {
  var fn = parse('lock[keys["aKey"]]');
  expect(fn({ keys: { aKey: 'theKey' }, lock: { theKey: 42 } })).toBe(42);
});
```

`lock[key]` 包含了四个 token：标识符 token `lock`，一个单字符 token `‘[’`，标识符 token `key`，还有一个单字符 token `‘]’`。它们在一起生成了一个 primary AST 节点。我们应该对 AST.primary 进行扩展，让它不仅支持接收点运算符，也接收左方括号：

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
  var next;
  while ((next = this.expect('.')) || (next = this.expect('['))) {
  //   primary = {
  //     type: AST.MemberExpression,
  //     object: primary,
  //     property: this.identifier()
  //   };
  // }
  // return primary;
};
```

要让可选的 `expect` 调用看起来简洁一些，我们可以对 `expect` 和 `peek` 进行扩展，让它们可以支持多个可选的 token。我们可以将这两个函数可以接收的参数扩展到 4 个：

_src/parse.js_

```js
AST.prototype.expect = function(e1, e2, e3, e4) {
  var token = this.peek(e1, e2, e3, e4);
//   if (token) {
//     return this.tokens.shift();
//   }
};
AST.prototype.peek = function(e1, e2, e3, e4) {
  // if (this.tokens.length > 0) {
  //   var text = this.tokens[0].text;
    if (text === e1 || text === e2 || text === e3 || text === e4 || (!e1 && !e2 && !e3 && !e4)) {
  //     return this.tokens[0];
  //   }
  // }
};
```

现在我们就可以在 `primary` 中使用传入两个参数的方式了：

_src/parse.js_

```js
// var next;
while ((next = this.expect('.', '['))) {
  // primary = {
  //   type: AST.MemberExpression,
  //   object: primary,
  //   property: this.identifier()
  // };
}
```

如果我们发现了当前处理字符是一个左方括号，那在表达式结束之前，我们还应该要消耗掉一个右方括号：

_src/parse.js_

```js
// var next;
while ((next = this.expect('.', '['))) {
//   primary = {
//     type: AST.MemberExpression,
//     object: primary,
//     property: this.identifier()
//   };
  if (next.text === '[') {
    this.consume(']');
  }
}
```

同时，我们在单元测试中也可以发现，方括号之间的内容不是标识符。这里面的内容其实是另一个 primary 表达式。实际上，要支持对这个内容的解析，我们最好还是把计算属性和非计算属性方法分开处理：

_src/parse.js_

```js
// var next;
while ((next = this.expect('.', '['))) {
  if (next.text === '[') {
    primary = {
      type: AST.MemberExpression,
      object: primary,
      property: this.primary()
    };
    this.consume(']');
  } else {
    // primary = {
    //   type: AST.MemberExpression,
    //   object: primary,
    //   property: this.identifier()
    // };
  }
}
```

AST 编译器也需要知道它现在处理的是计算属性访问还是非计算属性访问。在我们完成对 AST 构建器的修改之前，让我们把相关信息加入到 AST 节点中：

_src/parse.js_

```js
// var next;
while ((next = this.expect('.', '['))) {
  if (next.text === '[') {
  //   primary = {
  //     type: AST.MemberExpression,
  //     object: primary,
  //     property: this.primary(),
      computed: true
    // };
    // this.consume(']');
  } else {
    primary = {
      // type: AST.MemberExpression,
      // object: primary,
      // property: this.identifier(),
      computed: false
    };
  }
}
```
### 非计算属性查找

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

在 AST 编译器中，我们已经拥有了所需的所有部件了。我们要新增的是非计算成员的检索表达式，这个表达式的左边是类型为 `object` AST 节电，右边则是该节点的 `property`。

就像我们在简单的对象查找所做的那样，我们需要使用 `if` 语句来防止左侧的对象不存在。我们需要重用 `intoId` 这个变量名称，所以现在 `recurse` 函数的最上层声明好这个变量，一遍我们在新增的 `MemberExpression` 中使用：

_src/parse.js_

```js
ASTCompiler.prototype.recurse = function(ast) {
  var intoId;
  // switch (ast.type) {
  //   case AST.Program:
  //     this.state.body.push('return ', this.recurse(ast.body), ';');
  //     break;
  //   case AST.Literal:
  //     return this.escape(ast.value);
  //   case AST.ArrayExpression:
  //     var elements = _.map(ast.elements, _.bind(function(element) {
  //       return this.recurse(element);
  //     }, this));
  //     return '[' + elements.join(',') + ']';
  //   case AST.ObjectExpression:
  //     var properties = _.map(ast.properties, _.bind(function(property) {
  //       var key = property.key.type === AST.Identifier ?
  //         property.key.name :
  //         this.escape(property.key.value);
  //       var value = this.recurse(property.value);
  //       return key + ':' + value;
  //     }, this));
  //     return '{' + properties.join(',') + '}';
  //   case AST.Identifier:
      intoId = this.nextId();
  //     this.if_('s', this.assign(intoId, this.nonComputedMember('s', ast.name)));
  //     return intoId;
  //   case AST.ThisExpression:
  //     return 's';
    case AST.MemberExpression:
      intoId = this.nextId();
      var left = this.recurse(ast.object);
      this.if_(left,
        this.assign(intoId, this.nonComputedMember(left, ast.property.name)));
      return intoId;
  // }
};
```

有时你会需要对拥有两层以上的嵌套属性进行访问。一个 4 段的标识符查找应该与 2 段的标识符同样有效：

_test/parse_spec.js_

```js
it('looks up a 4-part identifier path from the scope', function() {
  var fn = parse('aKey.secondKey.thirdKey.fourthKey');
  expect(fn({ aKey: { secondKey: { thirdKey: { fourthKey: 42 } } } })).toBe(42);
  expect(fn({ aKey: { secondKey: { thirdKey: {} } } })).toBeUndefined();
  expect(fn({ aKey: {} })).toBeUndefined();
  expect(fn()).toBeUndefined();
});
```

无论需要访问在多少层的属性，它们都是一个 primary 表达式的一部分。解决这个问题的技巧就是吧 `AST.primary` 中的 `if` 语句转变为 `while` 语句。只要在表达式后还存在点运算符，我们就应该生成新的成员属性查找代码。前面一个成员查找的结果总会成为下一个成员查找的目标对象：

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
  while (this.expect('.')) {
  //   primary = {
  //     type: AST.MemberExpression,
  //     object: primary,
  //     property: this.identifier()
  //   };
  // }
  // return primary;
};
```

在继续之前，我们先来花点时间来分析一下这里面发生了什么，因为 AST 构建器和编译器中的递归代码已经非常强大了。针对表达式 `aKey.secondKey.thirdKey.fourthKey` 生成的抽象语法树会像下面这样：

```js
{
  type: AST.Program,
  body: {
    type: AST.MemberExpression,
    property: { type: AST.Identifier, name: 'fourthKey' },
    object: {
      type: AST.MemberExpresion,
      property: { type: AST.Identifier, name: 'thirdKey' },
      object: {
        type: AST.MemberExpression,
        property: { type: AST.Identifier, name: 'secondKey' },
        object: { type: AST.Identifier, name: 'aKey' }
      }
    }
  }
}
```

而 AST 编译器会把上述代码转换为一下的 JavaScript 函数：

```js
function(s) {
  var v0, v1, v2, v3;
  if (s) {
    v3 = s.aKey;
  }
  if (v3) {
    v2 = v3.secondKey;
  }
  if (v2) {
    v1 = v2.thirdKey;
  }
  if (v1) {
    v0 = v1.fourthKey;
  }
  return v0;
}
```

很整洁，不是吗？
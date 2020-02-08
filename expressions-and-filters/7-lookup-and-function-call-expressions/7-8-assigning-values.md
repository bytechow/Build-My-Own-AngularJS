### 赋值语句
#### Assigning Values

下面我们准备来看看表达式除了访问 scope 上的数据，还是如何使用赋值语句往 scope 上_放置_（put）数据的。举例来说，在表达式中把 scope 属性设置为某个值是完全合法的：

_test/parse_spec.js_

```js
it('parses a simple attribute assignment', function() {
  var fn = parse('anAttribute = 42');
  var scope = {};
  fn(scope);
  expect(scope.anAttribute).toBe(42);
});
```

要进行赋值的值也不一定只是一个简单的字面量。它也可以是一个 primary 表达式，比如函数调用：

_test/parse_spec.js_

```js
it('can assign any primary expression', function() { var fn = parse('anAttribute = aFunction()');
  var scope = {aFunction: _.constant(42)}; fn(scope);
  expect(scope.anAttribute).toBe(42);
});
```

除了简单的标识符，我们还可以使用计算属性和非计算属性进行赋值，还支持它们之间的组合：

_test/parse_spec.js_

```js
it('can assign a computed object property', function() {
  var fn = parse('anObject["anAttribute"] = 42');
  var scope = { anObject: {} };
  fn(scope);
  expect(scope.anObject.anAttribute).toBe(42);
});

it('can assign a non-computed object property', function() {
  var fn = parse('anObject.anAttribute = 42');
  var scope = { anObject: {} };
  fn(scope);
  expect(scope.anObject.anAttribute).toBe(42);
});

it('can assign a nested object property', function() {
  var fn = parse('anArray[0].anAttribute = 42');
  var scope = { anArray: [{}] };
  fn(scope);
  expect(scope.anArray[0].anAttribute).toBe(42);
});
```

就跟本章提到的大部分新特性一样，我们会需要先让 Lexer 能抛出一个 token 给 AST 构建器使用。这一次我们要抛出的是表示赋值语句的 `=` 号：

_src/parse.js_

```js
} else if (this.is('[],{}:.()=')) {
  this.tokens.push({ 
    text: this.ch
  });
  this.index++;
```

赋值语句跟我们之前提到的 primary AST 节点不一样，它跟我们之前看到的大部分节点都不同，它对应的 AST 不会在已经存在的 `AST.primary` 中存在。它会有自己的一个专属处理函数，叫 `assignment`。在这个函数里，我们会先左侧的 token，它会是一个 primary 节点，紧接着是一个等号 token，最后是右边的部分，是另一个 primary 节点：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.primary();
  if (this.expect('=')) {
    var right = this.primary();
  }
  return left;
};
```

左侧和右侧的子表达式合并在一次，就变成一个 `AssignmentExpression` 节点了：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  // var left = this.primary();
  // if (this.expect('=')) {
  //   var right = this.primary();
    return { type: AST.AssignmentExpression, left: left, right: right };
  // }
  // return left;
};
```

我们还需要先定义好 `AssignmentExpression` 这个常量：

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
AST.AssignmentExpression = 'AssignmentExpression';
```

但在 AST 构建器中，`assignment` 现在还是一个 “孤儿” 方式：它还没被调用过。

注意我们构建 `assignment` 的方式，我们是如何检测左侧是否紧跟着等号。如果后面并没有跟着一个等号，就会直接返回左侧部分。这意味着我们使用 `assignment` 可以构建一个赋值表达式，也可能会返回一个 primary 表达式。这种一旦尝试生成某种表达式失败后，也能回退到其他表达式的模式将会在下面的章节中经常看到。现在，我们可以在 `program` 中直接调用 `assignment` 来代替 `primary` 节点，这样就能处理现有所有的表达式了，无论是赋值表达式还是其他东西：

_src/parse.js_

```js
AST.prototype.program = function() {
  return {type: AST.Program, body: this.assignment()};
};
```

赋值表达式也能作为数组元素，因此我们也需要在数组表达式中替换相应方法：

_src/parse.js_

```js
AST.prototype.arrayDeclaration = function() {
  // var elements = [];
  // if (!this.peek(']')) {
  //   do {
  //     if (this.peek(']')) {
  //       break;
  //     }
      elements.push(this.assignment());
  //   } while (this.expect(','));
  // }
  // this.consume(']');
  // return { type: AST.ArrayExpression, elements: elements };
};
```

同样，我们也要对对象表达式的方法进行修改——但不包括对象的键，因为键一定是一个字符串：

_src/parse.js_

```js
AST.prototype.object = function() {
  // var properties = [];
  // if (!this.peek('}')) {
  //   do {
  //     var property = { type: AST.Property };
  //     if (this.peek().identifier) {
  //       property.key = this.identifier();
  //     } else {
  //       property.key = this.constant();
  //     }
  //     this.consume(':');
      property.value = this.assignment();
  //     properties.push(property);
  //   } while (this.expect(','));
  // }
  // this.consume('}');
  // return { type: AST.ObjectExpression, properties: properties };
};
```
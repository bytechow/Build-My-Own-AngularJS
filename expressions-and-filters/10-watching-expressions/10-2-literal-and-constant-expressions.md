### 字面量与常量表达式
#### Literal And Constant Expressions

现在我们已经知道解析器是如何返回一个用于计算原始表达式的函数的了。已返回的函数并不仅仅是一个普通函数，它需要添加一些额外的属性：

- `literal` —— 一个布尔值，用于说明表达式是否为字面量，例如整数或者数组字面量。
- `constant` —— 一个布尔值，用于说明表达式是否为一个常量（例如一个字面量原始类型），或者一个字面亮常量的集合。如果表达式是常量，它将永远保持不变。

比如，`42` 既是一个字面量，也是一个常量，`[42, 'abc']` 也是一样的。而像 `[42, 'abc', aVariable]` 这种虽然是一个字面量，但由于 `aVariable` 是一个变量，所以这个表达式就不是一个常量了。

`$parse` 服务的使用者有时会使用这两个标志来决定如何使用表达式。本章将会利用 `constant` 标志来对表达式监听做一些优化。

> 实际上，解析器返回的函数还有第三个属性 `assign`，本章后面会讲到。

我们会先介绍更容易实现的 `literal` 标识。所有简单的原始类型字面量，包括数字、字符串和布尔值，都会被标记为字面量：

_test/parse_spec.js_

```js
it('marks integers literal', function() {
  var fn = parse('42');
  expect(fn.literal).toBe(true);
});

it('marks strings literal', function() {
  var fn = parse('"abc"');
  expect(fn.literal).toBe(true);
});

it('marks booleans literal', function() {
  var fn = parse('true');
  expect(fn.literal).toBe(true);
});
```

数组和对象字面量也一样：

_test/parse_spec.js_

```js
it('marks arrays literal', function() {
  var fn = parse('[1, 2, aVariable]');
  expect(fn.literal).toBe(true);
});

it('marks objects literal', function() {
  var fn = parse('{a: 1, b: aVariable}');
  expect(fn.literal).toBe(true);
});
```

其它值都会被标记为“非字面量”（non-literal）：

_test/parse_spec.js_

```js
it('marks unary expressions non-literal', function() {
  var fn = parse('!false');
  expect(fn.literal).toBe(false);
});

it('marks binary expressions non-literal', function() {
  var fn = parse('1 + 2');
  expect(fn.literal).toBe(false);
});
```

我们需要使用一个辅助函数 `isLiteral` 来检查 AST 是否是一个字面量。然后我们会将检查结果作为编译后的表达式函数的一个属性：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = {
  //   body: [],
  //   nextId: 0,
  //   vars: [],
  //   filters: {}
  // };
  // this.recurse(ast);
  // var fnString = this.filterPrefix() +
  //   'var fn=function(s,l){' + (this.state.vars.length ?
  //     'var ' + this.state.vars.join(',') + ';' :
  //     ''
  //   ) + this.state.body.join('') + '}; return fn;';
  /* jshint -W054 */
  var fn = new Function(
    // 'ensureSafeMemberName',
    // 'ensureSafeObject',
    // 'ensureSafeFunction',
    // 'ifDefined',
    // 'filter',
    // fnString)(
    //   ensureSafeMemberName,
    //   ensureSafeObject,
    //   ensureSafeFunction,
    //   ifDefined,
    //   filter);
  /* jshint +W054 */
  fn.literal = isLiteral(ast);
  return fn;
};
```

`isLiteral` 函数的定义如下：

- 如果程序（program）为空则认为是字面量表达式
- 如果程序非空，且它仅有的表达式是一个字面量、
数组或者对象，也认为是一个字面量表达式

翻译成代码就是：

_src/parse.js_

```js
function isLiteral(ast) {
  return ast.body.length === 0 ||
    ast.body.length === 1 && (
      ast.body[0].type === AST.Literal ||
      ast.body[0].type === AST.ArrayExpression ||
      ast.body[0].type === AST.ObjectExpression);
}
```

设置 `constant` 标识的过程会更复杂一点。我们需要分别考虑每一种 AST 节点类型，以了解如何确定它的“不变性”（constantness）。

我们先从简单的字面量开始。数字、字符串和布尔值都是常量：

_test/parse_spec.js_

```js
it('marks integers constant', function() {
  var fn = parse('42');
  expect(fn.constant).toBe(true);
});

it('marks strings constant', function() {
  var fn = parse('"abc"');
  expect(fn.constant).toBe(true);
});

it('marks booleans constant', function() {
  var fn = parse('true');
  expect(fn.constant).toBe(true);
});
```

跟 `literal` 标识属性类似，这些原始类型生成的函数会有一个 `constant` 标识属性。我们会从 AST 的根节点读取这个标识对应的值：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = {
  //   body: [],
  //   nextId: 0,
  //   vars: [],
  //   filters: {}
  // };
  // this.recurse(ast);
  // var fnString = this.filterPrefix() +
  //   'var fn=function(s,l){' + (this.state.vars.length ?
  //     'var ' + this.state.vars.join(',') + ';' :
  //     ''
  //   ) + this.state.body.join('') + '}; return fn;';
  // /* jshint -W054 */
  // var fn = new Function(
  //   'ensureSafeMemberName',
  //   'ensureSafeObject',
  //   'ensureSafeFunction',
  //   'ifDefined',
  //   'filter',
  //   fnString)(
  //     ensureSafeMemberName,
  //     ensureSafeObject,
  //     ensureSafeFunction,
  //     ifDefined,
  //     filter);
  // /* jshint +W054 */
  // fn.literal = isLiteral(ast);
  fn.constant = ast.constant;
  // return fn;
};
```

但问题是 AST 根节点现在还没有这个标识属性。那怎么办呢？我们要做的是在编译之前先使用一个叫 `markConstantExpressions` 的函数对 AST 进行预处理。在这个预处理过程将会设置 `constant` 标识：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  markConstantExpressions(ast);
  // this.state = {
  //   body: [],
  //   nextId: 0,
  //   vars: [],
  //   filters: {}
  // };
  // this.recurse(ast);
  // var fnString = this.filterPrefix() +
  //   'var fn=function(s,l){' + (this.state.vars.length ?
  //     'var ' + this.state.vars.join(',') + ';' :
  //     ''
  //   ) + this.state.body.join('') + '}; return fn;';
  // /* jshint -W054 */
  // var fn = new Function(
  //   'ensureSafeMemberName',
  //   'ensureSafeObject',
  //   'ensureSafeFunction',
  //   'ifDefined',
  //   'filter',
  //   fnString)(
  //     ensureSafeMemberName,
  //     ensureSafeObject,
  //     ensureSafeFunction,
  //     ifDefined,
  //     filter);
  // /* jshint +W054 */
  // fn.literal = isLiteral(ast);
  // fn.constant = ast.constant;
  // return fn;
};
```

跟 `recurse` 函数相似之处在于，`markConstantExpressions` 也是一个递归函数，函数体中包含一个篇幅较长的 `switch` 语句。举个例子，字面量表达式都会是常量，所以当 `markConstantExpressions` 发现传入的是一个类型为字面量 AST 节点，它就会把 AST 的 `constant`  标识属性设置为 `true`：

_src/parse.js_

```js
function markConstantExpressions(ast) {
  switch (ast.type) {
    case AST.Literal:
      ast.constant = true;
      break;
  }
}
```

在让测试用例通过之前，我们还需要考虑 AST 根节点总会是 `Program` 类型的这一事实。一个程序（program）可能包含多个子表达式。当 `markConstantExpressions` 发现这是一个 program 节点时，它需要递归调用自身以便处理它下面的所有子表达式：

_src/parse.js_

```js
function markConstantExpressions(ast) {
  // switch (ast.type) {
    case AST.Program:
      _.forEach(ast.body, function(expr) {
        markConstantExpressions(expr);
      });
      break;
  //   case AST.Literal:
  //     ast.constant = true;
  //     break;
  // }
}
```

如果一个 `Program` 节点下面所有的子节点都是常量，那它就可以被标记为是常量了：

_src/parse.js_

```js
function markConstantExpressions(ast) {
  var allConstants;
  // switch (ast.type) {
  //   case AST.Program:
      allConstants = true;
      // _.forEach(ast.body, function(expr) {
      //   markConstantExpressions(expr);
        allConstants = allConstants && expr.constant;
      // });
      ast.constant = allConstants;
  //     break;
  //   case AST.Literal:
  //     ast.constant = true;
  //     break;
  // }
}
```

标识符表达式肯定不是一个常量——我们并不知道它之后是否会发生改变：

_test/parse_spec.js_

```js
it('marks identifiers non-constant', function() {
  var fn = parse('a');
  expect(fn.constant).toBe(false);
});
```

实现起来也非常简单：

_src/parse.js_

```js
case AST.Identifier:
  ast.constant = false;
  break;
```

当且仅当一个数组表达式的所有元素都是常量时，这个数组才是一个常量：

_test/parse_spec.js_

```js
it('marks arrays constant when elements are constant', function() {
  expect(parse('[1, 2, 3]').constant).toBe(true);
  expect(parse('[1, [2, [3]]]').constant).toBe(true);
  expect(parse('[1, 2, a]').constant).toBe(false);
  expect(parse('[1, [2, [a]]]').constant).toBe(false);
});
```

我们检查数组的方式跟处理 program 差不多：我们需要对数组进行递归处理，然后根据实际情况标上标识：

_src/parse.js_

```js
case AST.ArrayExpression:
  allConstants = true;
  _.forEach(ast.elements, function(element) {
    markConstantExpressions(element);
    allConstants = allConstants && element.constant;
  });
  ast.constant = allConstants;
  break;
```

对象也一样，对象是否是常量，取决于对象里的值是否都为常量。（对象的键只会是字符串，本来就是常量，因此不需要考虑。）

_test/parse_spec.js_

```js
it('marks objects constant when values are constant', function() {
  expect(parse('{a: 1, b: 2}').constant).toBe(true);
  expect(parse('{a: 1, b: {c: 3}}').constant).toBe(true);
  expect(parse('{a: 1, b: something}').constant).toBe(false);
  expect(parse('{a: 1, b: {c: something}}').constant).toBe(false);
});
```

我们同样通过递归便利对象属性来确定对象的常量标识是什么：

_src/parse.js_

```js
case AST.ObjectExpression:
  allConstants = true;
  _.forEach(ast.properties, function(property) {
    markConstantExpressions(property.value);
    allConstants = allConstants && property.value.constant;
  });
  ast.constant = allConstants;
  break;
```

`this` 并不是常量。它也不能是常量，因为它的值是运行时的 Scope（作用域）：

_test/parse_spec.js_

```js
it('marks this as non-constant', function() {
  expect(parse('this').constant).toBe(false);
});
```

那 `markConstantExpressions` 对 `ThisExpression` 和 `LocalsExpression` 两种节点的处理也就呼之欲出了：

_src/parse.js_

```js
case AST.ThisExpression:
case AST.LocalsExpression:
  ast.constant = false;
  break;
```

对于非计算型的属性查找表达式，它是否为常量取决于它要查找的那个对象是否为常量：

_test/parse_spec.js_

```js
it('marks non-computed lookup constant when object is constant', function() {
  expect(parse('{a: 1}.a').constant).toBe(true);
  expect(parse('obj.a').constant).toBe(false);
});
```

当遇到属性查找表达式，我们可以先检查它的目标对象，再决定它对应的常量标识：

_src/parse.js_

```js
case AST.MemberExpression:
  markConstantExpressions(ast.object);
  ast.constant = ast.object.constant;
  break;
```

如果是计算型的属性查找，我们还需要看查找属性时使用的键（key）是不是一个常量：

_test/parse_spec.js_

```js
it('marks computed lookup constant when object and key are', function() {
  expect(parse('{a: 1}["a"]').constant).toBe(true);
  expect(parse('obj["a"]').constant).toBe(false);
  expect(parse('{a: 1}[something]').constant).toBe(false);
  expect(parse('obj[something]').constant).toBe(false);
});
```

如果是计算型的属性查找，我们需要额外检查 property 节点：

_src/parse.js_

```js
case AST.MemberExpression:
  // markConstantExpressions(ast.object);
  if (ast.computed) {
    markConstantExpressions(ast.property);
  }
  ast.constant = ast.object.constant &&
                  (!ast.computed || ast.property.constant);
  // break;
```

函数调用表达式就不是一个常量——我们不能对函数的性质作这样的假设：

_test/parse_spec.js_

```js
it('marks function calls non-constant', function() {
  expect(parse('aFunction()').constant).toBe(false);
});
```

在遇到函数调用表达式时，我们直接把常量标识设置为 `false` 就可以了：

_src/parse.js_

```js
case AST.CallExpression:
  ast.constant = false;
  break;
```
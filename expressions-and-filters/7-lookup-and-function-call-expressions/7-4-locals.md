### 本地参数
#### Locals

到目前为止，`parse` 返回的函数都只会接收一个参数——作用域。纯字面量表达式生成的函数甚至可以把这个函数忽略掉。

Angular 表达式实际上还能接受第二个参数，叫 `locals`。这个参数实际上也跟 `scope` 参数一样，只是一个对象而已，而这里的约束是，被解析的表达式不仅要从 `scope` 中进行属性查找，还要从 `locals` 对象里进行查找。我们会先在 `locals` 中进行查找，若在本地变量中查找不到的时候，才到 `scope` 中进行查找。

这意味着我们可以有效地使用 `locals` 参数对 `scope` 属性进行覆盖。换句话来说，我们可以访问到指定属性，即使这个属性没有在任何作用域中进行注册。这对于指令和它们的绑定数据（bingings）来说尤为有用。举例来说，`ngClick` 指令允许我们通过引用 `$event` 来访问到点击事件对象，这就是通过对表达式添加本地参数的方式来完成的。

_test/parse_spec.js_

```js
it('uses locals instead of scope when there is a matching key', function() {
  var fn = parse('aKey');
  var scope = { aKey: 42 };
  var locals = { aKey: 43 };
  expect(fn(scope, locals)).toBe(43);
});

it('does not use locals instead of scope when no matching key', function() {
  var fn = parse('aKey');
  var scope = { aKey: 42 };
  var locals = { otherKey: 43 };
  expect(fn(scope, locals)).toBe(42);
});
```

> 你可能还记得，我们已经在 `Scope.prototype.$eval` 中看到过类似的本地变量，`$eval` 会接受一个 `locals` 参数。实际上，这个参数会直接传递给表达式，我们会在之后把表达式和 watch 集成时看到这一点。

要注意本地参数与作用域的“竞争”规则只适用于键的第一部分。如果一个多层属性访问的第一层是与 `locals` 的第一层键匹配的话，就结束查找当前键的过程（认为这个表达式应当在 `locals` 中进行属性查找），即使第二层键没有发现匹配项：

_test/parse_spec.js_

```js
it('uses locals instead of scope when the first part matches', function() {
  var fn = parse('aKey.anotherKey');
  var scope = { aKey: { anotherKey: 42 } };
  var locals = { aKey: {} };
  expect(fn(scope, locals)).toBeUndefined();
});
```

我们生成的 JavaScript 函数现在需要接收两个参数，第二个参数就是 locals 对象。在生成的代码中，locals 对象会被称作 `l`：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = { body: [], nextId: 0, vars: [] };
  // this.recurse(ast);
  /* jshint -W054 */
  return new Function('s', 'l',
    // (this.state.vars.length ?
    //   'var ' + this.state.vars.join(',') + ';' : ''
    // ) + this.state.body.join('')); 
  /* jshint +W054 */
};
```

在编译器内部，我们需要修改的相关模块是 `AST.Identifier` 生成的代码，`AST.Identifier` 现在会对作用域属性进行查找。我们需要加入一个额外的判断，看看我们是要从 locals 中进行查找，还是从 scope（作用域）中进行查找：

_src/parse.js_

```js
case AST.Identifier:
  // intoId = this.nextId();
  this.if_('l',
    this.assign(intoId, this.nonComputedMember('l', ast.name)));
  this.if_(this.not('l') + ' && s',
    // this.assign(intoId, this.nonComputedMember('s', ast.name)));
  // return intoId;
```

现在我们会先检查一下 `l` 是否存在，如果没有找到 `l`，我们才从 `s` 中查找标识符。`not` 方法是新的，所以先定义好。其实它只是把 JavaScript 表达式的值进行取反而已：

_src/parse.js_

```js
ASTCompiler.prototype.not = function(e) {
  return '!(' + e + ')';
};
```

我们的测试用例现在还没有通过，问题出在我们查找 locals 属性的方式上。按照约定，只有在属性确实存在于 locals 才能被访问，但现在是只要 locals 存在，我们都会使用 locals 上的属性。我们应该要检查一下，看标识符上是否真的有属性匹配这个标识符，我们可以借助一个叫 `getHasOwnProperty` 的方法来实现这个目标：

_src/parse.js_

```js
case AST.Identifier:
  // intoId = this.nextId();
  this.if_(this.getHasOwnProperty('l', ast.name),
    // this.assign(intoId, this.nonComputedMember('l', ast.name)));
  this.if_(this.not(this.getHasOwnProperty('l', ast.name)) + ' && s',
    // this.assign(intoId, this.nonComputedMember('s', ast.name)));
  // return intoId;
```

`getHasOwnProperty` 方法会接收一个对象和一个属性名称。它首先会检查这个对象是否存在，然后看这个对象里面是否确实存在这个属性。要确认后者，我们需要借助 JavaScript 的 [in 运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/in)：

_src/parse.js_

```js
ASTCompiler.prototype.getHasOwnProperty = function(object, property) { return object + '&&(' + this.escape(property) + ' in ' + object + ')';
};
```

除了能查找 locals 对象中的单个属性以外，表达式解析器还提供了一个特殊的 `$locals` 名称让我们可以访问到整个 locals 对象。当你需要验证 locals 数据的可用性的时候，这个功能会很有用：

_test/parse_spec.js_

```js
it('will parse $locals', function() {
  var fn = parse('$locals');
  var scope = {};
  var locals = {};
  expect(fn(scope, locals)).toBe(locals);
  expect(fn(scope)).toBeUndefined();
  
  fn = parse('$locals.aKey');
  scope = { aKey: 42 };
  locals = { aKey: 43 };
  expect(fn(scope, locals)).toBe(43);
});
```

为此，我们需要新增一个 AST token：

_src/parse.js_

```js
// AST.Program = 'Program’;
// AST.Literal = ‘Literal’;
// AST.ArrayExpression = ‘ArrayExpression’;
// AST.ObjectExpression = ‘ObjectExpression’;
// AST.Property = ‘Property’;
// AST.Identifier = ‘Identifier’;
// AST.ThisExpression = ‘ThisExpression’;
AST.LocalsExpression = 'LocalsExpression';
// AST.MemberExpression = ‘MemberExpression’;
```

跟 `this` 表达式类似，`$locals` 表达式也是一个常量，所以我们需要把它加入到当前存储常量的对象中去：

_src/parse.js_

```js
AST.prototype.constants = {‘
  // null’: { type: AST.Literal, value: null },
  // ‘true’: { type: AST.Literal, value: true },
  // ‘false’: { type: AST.Literal, value: false },
  // ‘this’: { type: AST.ThisExpression },
  '$locals': { type: AST.LocalsExpression }
};
```

在 `ASTCompiler.recurse` 中，如果遇到这个 token，我们可以直接返回表达式函数中的 `l` 参数：

_src/parse.js_

```js
case AST.LocalsExpression:
  return 'l';
```
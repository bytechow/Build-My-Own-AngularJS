### 简单的属性查找
#### Simple Attribute Lookup

访问作用域属性最简单的方式就是按属性名称检索：表达式 `‘aKey’` 会从作用域对象中找到 `aKey` 属性，然后返回它：

_test/parse_spec.js_

```js
it('looks up an attribute from the scope', function() {
  var fn = parse('aKey');
  expect(fn({aKey: 42})).toBe(42);
  expect(fn({})).toBeUndefined();
});
```

注意，`parse` 函数的返回值是一个函数，这个函数会接受一个 JavaScript 对象作为参数。这个对象一般都是 `Scope` 的实例，这个实例能被表达式访问或操作。但它不一定非得是一个作用域，在单元测试中，我们可以直接用普通的对象字面量。由于字面量表达式跟作用域没有半点关系，之前我们都没有使用过这个参数，但在本章情况会发生变化。事实上，我们要做的第一个改变就是在嘴中编译生成的表达式函数中加入这个参数。我们把它称为 `s`：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  var ast = this.astBuilder.ast(text);
  this.state = { body: [] };
  this.recurse(ast);
  /* jshint -W054 */
  return new Function('s', this.state.body.join(''));
  /* jshint +W054 */
};
```

在解析时，表达式 `‘aKey’` 会被转化为一个标识符 token 还有一个类型为 `Identifier` AST 节点。之前，我们已经把标识符节点应用在对象的键上了。下一步，我们会扩展标识符节点，让它也支持属性查找。

在 AST 构建器中，当我们在构建 primary AST 节点时，需要加入对标识符的检测。我们会在构建对象属性节点时做同样的事情：如果找到了一个标识符 token，就创建一个 `Identifier` 节点。

_src/parse.js_

```js
AST.prototype.primary = function() {
  // if (this.expect('[')) {
  //   return this.arrayDeclaration();
  // } else if (this.expect('{')) {
  //   return this.object();
  // } else if (this.constants.hasOwnProperty(this.tokens[0].text)) {
  //   return this.constants[this.consume().text];
  } else if (this.peek().identifier) {
    return this.identifier();
  // } else {
  //   return this.constant();
  // }
};
```

同样地，AST 编译器现在需要能在 `recurse` 方法处理标识符。如果发现标识符，就把它当作作用域上的属性成员进行处理——也就是是 `s`。

_src/parse.js_

```js
case AST.Identifier:
  return this.nonComputedMember('s', ast.name);
```

`nonComputedMember` 方法会接收两个参数：一个是要被查找的对象，另一个是要查找的成员属性。它会为 `a.b` 这种非计算的属性成员生成 JavaScript 代码（跟 `a[b]` 这种计算属性的查找做对比）：

_src/parse.js_

```js
ASTCompiler.prototype.nonComputedMember = function(left, right) {
  return '(' + left + ').' + right;
};
```

这样就通过了我们的单元测试了：我们现在正在把 `‘aKey’` 这种表达式变成函数体中的 `return s.aKey`。现在我们的表达式语言立马就比之前有用了许多，特别是对于经常要做属性查找的 watch 表达式。

如果你之前已经编写过 AngularJS 应用，你可能注意到 Angular 表达式在无法找到属性时也非常宽容。跟 JavaScript 不一样，Angular 表达式即使在访问属性时，无法找到目标对象，也不会抛出异常。举个例子来说，如果我们在运算表达式函数时没有传入任何参数，`s` 的值为 `undefined`，不会抛出任何异常：

_test/parse_spec.js_

```js
it('returns undefined when looking up attribute from undefined', function() {
  var fn = parse('aKey');
  expect(fn()).toBeUndefined();
});
```

这就意味着，我们需要在 JavaScript 代码中加入判断，首先检查一下 `s` 是否真的存在，之后才在这个对象中做属性访问。实际上，我们要将 `aKey` 表达式转化为类似下面这样的代码：

```js
function(s) {
  var v0;
  if (s) {
    v0 = s.aKey;
  }
  return v0;
}
```

要加入 if 语句，我们需要增加一个叫 `if_` 的方法，这个方法会接收两个参数：一个是测试表达式，另一个是在满足测试表达式后要执行的语句。它会生成相应的 JavaScript `if` 语句，然后把它加入到表达式的内容中去：

_src/parse.js_

```js
ASTCompiler.prototype.if_ = function(test, consequent) {
  this.state.body.push('if(', test, '){', consequent, '}');
};
```

在标识符的查找过程中国呢，我们将用它来检查 `s` 是否存在：

_src/parse.js_

```js
case AST.Identifier:
  this.if_('s', '');
  // return this.nonComputedMember('s', ast.name);
```

下一步，我们需要在 `if` 代码块的前面加入一个变量，并在 if 代码块中对作用域属性进行赋值，最后在这个 `recurse` 调用时返回这个变量的值：

_src/parse.js_

```js
case AST.Identifier:
  this.state.body.push('var v0;');
  this.if_('s', 'v0=' + this.nonComputedMember('s', ast.name) + ';');
  return 'v0';
```

这样我们的测试就通过了，但在继续之前，让我们先花一点时间重构代码，让代码更容易进行扩展。例如，我们可以引入一个帮助函数来进行变量赋值：

_src/parse.js_

```js
ASTCompiler.prototype.assign = function(id, value) {
  return id + '=' + value + ';';
};
```

我们会在 `if` 语句中使用这个赋值语句：

_src/parse.js_

```js
case AST.Identifier:
  // this.state.body.push('var v0;');
  this.if_('s', this.assign('v0', this.nonComputedMember('s', ast.name)));
  // return 'v0';
```

当然，还有很多表达式是要用到多个变量，生成变量名称时要保证不产生冲突是比较困难。为此，我们要引入一个运行中的计数器，它是构成唯一 ID 的基础。我们会把它初始化为 0：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  this.state = { body: [], nextId: 0 };
  // this.recurse(ast);
  // /* jshint -W054 */
  // return new Function('s', this.state.body.join('')); /* jshint +W054 */
};
```

然后我们也会创建一个叫 `nextId` 的方法，它会生成一个变量名称，同时让计数器自增：

_src/parse.js_

```js
ASTCompiler.prototype.nextId = function() {
  var id = 'v' + (this.state.nextId++);
  return id;
};
```

然后我们会在检索标识符使用这个函数：

_src/parse.js_

```js
case AST.Identifier:
  var intoId = this.nextId();
  this.state.body.push('var ', intoId, ';');
  this.if_('s', this.assign(intoId, this.nonComputedMember('s', ast.name)));
  return intoId;
```

最后，`var` 声明不应当真的成为标识符查找过程的一部分。在 JavaScript 中，变量声明会[自动提升到函数的最前面](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)，那如果我们一开始就这样做的话不就更好了吗。所以，我们会在编译状态中引入 `vars` 数组：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  this.state = { body: [], nextId: 0, vars: [] };
  // this.recurse(ast);
  //  jshint -W054 
  // return new Function('s', this.state.body.join('')); /* jshint +W054 */
};
```

每次调用 `nextId` 的时候，就会生成一个变量名称：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = { body: [], nextId: 0, vars: [] };
  // this.recurse(ast);
  /* jshint -W054 */
  return new Function('s', (this.state.vars.length ?
    'var ' + this.state.vars.join(',') + ';' :
    ''
  ) + this.state.body.join(''));
  /* jshint +W054 */
};
```

现在，我们不再需要在处理标识符的分支加入 `var` 语句了：

_src/parse.js_

```js
case AST.Identifier:
  var intoId = this.nextId();
  this.if_('s', this.assign(intoId, this.nonComputedMember('s', ast.name)));
  return intoId;
```

现在我们就有了一个可以在表达式函数中引入任意数量变量的机制。这种机制在之后会变得非常有用。
### 在外部进行赋值
#### External Assignment

在对表达式进行求值时，重点通常是获取它们在作用域上的当前值。这就是 Angular 中整个数据绑定系统的基础。

在某些情况下，我们需要一种不同的调用表达式的方式。那就是在作用域范围内为它们赋予一个新值。我们会通过在表达式函数暴露一个 `assign` 方法来实现这种模式。

举个例子：如果你有一个形如 `a.b` 的表达式。你可以在作用域上调用它：

```js
var exprFn = parse('a.b');
var scope = {a: {b: 42}};
exprFn(scope); // => 42
```

但你也可以在作用域上对它进行赋值，也就是说，我们可以直接在给定的作用域上替换表达式对应的值：

```js
var exprFn = parse('a.b');
var scope = { a: { b: 42 } };
exprFn.assign(scope, 43);
scope.a.b; // => 43
```

在本书后面的章节中，我们将使用 `assign` 方法为隔离作用域实现双向绑定。

这本质上是作为函数对外暴露的 `AssignmentExpression` 的逻辑。使用单元测试表示出来就是，我们既可以对简单的变量标识符进行赋值，也可以对内嵌的属性成员进行赋值：

_test/parse_spec.js_

```js
it('allows calling assign on identifier expressions', function() {
  var fn = parse('anAttribute');
  expect(fn.assign).toBeDefined();

  var scope = {};
  fn.assign(scope, 42);
  expect(scope.anAttribute).toBe(42);
});

it('allows calling assign on member expressions', function() {
  var fn = parse('anObject.anAttribute');
  expect(fn.assign).toBeDefined();
  
  var scope = {};
  fn.assign(scope, 42);
  expect(scope.anObject).toEqual({ anAttribute: 42 });
});
```

我们要做的就是再生成一个表达式函数，然后把它变成主表达式函数的 `assign` 方法。它会有自己的编译状态和编译阶段：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // markConstantAndWatchExpressions(ast);
  // this.state = {
  //   nextId: 0,
  //   fn: { body: [], vars: [] },
  //   filters: {},
    assign: { body: [], vars: [] },
  //   inputs: []
  // };
  // this.stage = 'inputs';
  // _.forEach(getInputs(ast.body), _.bind(function(input, idx) {
  //   var inputKey = 'fn' + idx;
  //   this.state[inputKey] = { body: [], vars: [] };
  //   this.state.computing = inputKey;
  //   this.state[inputKey].body.push('return ' + this.recurse(input) + ';');
  //   this.state.inputs.push(inputKey);
  // }, this));
  this.stage = 'assign';
  
  // this.stage = 'main';
  // this.state.computing = 'fn';
  // this.recurse(ast);
  // ...
};
```

只有当我们可以为表达式生成一个“可赋值的抽象语法树”（assignable AST）时，才会生成 assign 方法：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // markConstantAndWatchExpressions(ast);
  // this.state = {
  //   nextId: 0,
  //   fn: { body: [], vars: [] },
  //   filters: {},
  //   assign: { body: [], vars: [] },
  //   inputs: []
  // };
  // this.stage = 'inputs';
  // _.forEach(getInputs(ast.body), _.bind(function(input, idx) {
  //   var inputKey = 'fn' + idx;
  //   this.state[inputKey] = { body: [], vars: [] };
  //   this.state.computing = inputKey;
  //   this.state[inputKey].body.push('return ' + this.recurse(input) + ';');
  //   this.state.inputs.push(inputKey);
  // }, this));
  // this.stage = 'assign';
  var assignable = assignableAST(ast);
  if (assignable) {
    
  }
  // this.stage = 'main';
  // this.state.computing = 'fn';
  // this.recurse(ast);
  // ...
};
```

只有当表达式只有一条类型为标识符或成员的语句时，才会生成“可赋值的抽象语法树”：

_src/parse.js_

```js
function isAssignable(ast) {
  return ast.type === AST.Identifier || ast.type == AST.MemberExpression;
}

function assignableAST(ast) {
  if (ast.body.length == 1 && isAssignable(ast.body[0])) {
    
  }
}
```

可赋值的抽象语法书实际上是包装在 `AST.AssignmentExpression` 节点中的原始表达式。赋值表达式的右侧是一个特殊的值——它的类型是 `AST.NGValueParameter`，它表示在运行时提供的参数化值（parameterized value）。它本质上是值的占位符，只有在赋值发生时才会知道该值的具体内容：

_src/parse.js_

```js
function assignableAST(ast) {
  // if (ast.body.length == 1 && isAssignable(ast.body[0])) {
    return {
      type: AST.AssignmentExpression,
      left: ast.body[0],
      right: { type: AST.NGValueParameter }
    };
  // }
}
```

现在，我们（可能）已经有了“可赋值的抽象语法树”，我们就可以继续在 `recurse` 中对它进行编译：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // markConstantAndWatchExpressions(ast);
  // this.state = {
  //   nextId: 0,
  //   fn: { body: [], vars: [] },
  //   filters: {},
  //   assign: { body: [], vars: [] },
  //   inputs: []
  // };
  // this.stage = 'inputs';
  // _.forEach(getInputs(ast.body), _.bind(function(input, idx) {
  //   var inputKey = 'fn' + idx;
  //   this.state[inputKey] = { body: [], vars: [] };
  //   this.state.computing = inputKey;
  //   this.state[inputKey].body.push('return ' + this.recurse(input) + ';');
  //   this.state.inputs.push(inputKey);
  // }, this));
  // this.stage = 'assign';
  // var assignable = assignableAST(ast);
  // if (assignable) {
    this.state.computing = 'assign';
    this.state.assign.body.push(this.recurse(assignable));
  // }
  // this.stage = 'main';
  // this.state.computing = 'fn';
  // this.recurse(ast);
  // ...
};
```

我们将从编译结果中生成 `assign` 函数。它会接收是三个参数：一个作用域、要赋值的值和本地变量。我们会把这个函数的代码放到一个新变量 `extra` 中，然后把它也赋值到主表达式上去：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  var extra = '';
  // markConstantAndWatchExpressions(ast);
  // this.state = {
  //   nextId: 0,
  //   fn: { body: [], vars: [] },
  //   filters: {},
  //   assign: { body: [], vars: [] },
  //   inputs: []
  // };
  // this.stage = 'inputs';
  // _.forEach(getInputs(ast.body), _.bind(function(input, idx) {
  //   var inputKey = 'fn' + idx;
  //   this.state[inputKey] = { body: [], vars: [] };
  //   this.state.computing = inputKey;
  //   this.state[inputKey].body.push('return ' + this.recurse(input) + ';');
  //   this.state.inputs.push(inputKey);
  // }, this));
  // this.stage = 'assign';
  // var assignable = assignableAST(ast);
  // if (assignable) {
  //   this.state.computing = 'assign';
  //   this.state.assign.body.push(this.recurse(assignable));
    extra = 'fn.assign = function(s,v,l){' + (this.state.assign.vars.length ?
      'var ' + this.state.assign.vars.join(',') + ';' :
      ''
    ) + this.state.assign.body.join('') + '};';
  // }
  // this.stage = 'main';
  // this.state.computing = 'fn';
  // this.recurse(ast);
  // var fnString = this.filterPrefix() +
  //   'var fn=function(s,l){' + (this.state.fn.vars.length ?
  //     'var ' + this.state.fn.vars.join(',') + ';' :
  //     ''
  //   ) + this.state.fn.body.join('') + '};' +
  //   this.watchFns() +
    extra +
  //   ' return fn;';
  // /* jshint -W054 */
  // var fn = new Function(
  //   'ensureSafeMemberName', 'ensureSafeObject', 'ensureSafeFunction', 'ifDefined',
  //   'filter', fnString)(
  //   ensureSafeMemberName,
  //   ensureSafeObject,
  //   ensureSafeFunction,
  //   ifDefined,
  //   filter);
  // /* jshint +W054 */
  // fn.literal = isLiteral(ast);
  // fn.constant = ast.constant;
  // return fn;
};
```

现在我们唯一缺少的部分就是在 `recurse` 方法中编译 `NGValueParameter` 节点。它的工作内容就是在生成的代码中标记出要赋值给对外暴露的 `assign` 函数的参数位置。我们刚才在生成的代码中传递的参数是 `v`，因此我们可以直接将这个 AST 节点编译成 `v`，一切就准备就绪了：

_src/parse.js_

```js
case AST.NGValueParameter:
  return 'v';
```
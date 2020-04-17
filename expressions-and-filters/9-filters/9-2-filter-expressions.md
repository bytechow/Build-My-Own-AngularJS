### 过滤器表达式
#### Filter Expressions

现在我们已经做好了在表达式解释器开发过滤器的准备了。我们想要达到的目标是在表达式中使用管道符号改变表达式的最终结果。然后，我们要能提供之前注册过的过滤器名称，然后把表达式的值作为输入参数调用这个过滤器：

_test/parse_spec.js_

```js
it('can parse filter expressions', function() {
  register('upcase', function() {
    return function(str) {
      return str.toUpperCase();
    };
  });
  var fn = parse('aString | upcase');
  expect(fn({ aString: 'Hello' })).toEqual('HELLO');
});
```

我们需要在测试文件中引入 `register` 函数：

_test/parse_spec.js_

```js
'use strict';

var _ = require('lodash');
var parse = require('../src/parse');
var register = require('../src/filter').register;
```

由于我们需要把管道符号作为表达式中的一个运算符，因此，我们需要在 Lexer 支持的运算符列表中加入管道符号：

_src/parse.js_

```js
var OPERATORS = {
  '+': true,
  '-': true,
  '!': true,
  '*': true,
  '/': true,
  '%': true,
  '=': true,
  '==': true,
  '!=': true,
  '===': true,
  '!==': true,
  '<': true,
  '>': true,
  '<=': true,
  '>=': true,
  '&&': true,
  '||': true,
  '|': true
};
```

这样，当 Lexer 发现有单个管道符号出现时，会抛出这个符号：

下一步，我们将要为过滤器表达式构建一个 AST 节点。我们会在 AST 构建器中使用一个新方法 `filter` 来处理。它首先会处理左侧的赋值表达式（或者其他拥有更高优先级的表达式），然后看看这个表达式后面是否跟着一个管道字符：

_src/parse.js_

```js
AST.prototype.filter = function() {
  var left = this.assignment();
  if (this.expect('|')) {
    
  }
  return left;
};
```

如果有管道符号，我们就会创建一个 `CallExpression`。被调用函数就是过滤器的名称，我们会把它当作一个标识符节点来进行消耗。调用时要传入唯一的参数就是之前左侧表达式的结果：

_src/parse.js_

```js
AST.prototype.filter = function() {
  // var left = this.assignment();
  // if (this.expect('|')) {
    left = {
      type: AST.CallExpression,
      callee: this.identifier(),
      arguments: [left]
    };
  // }
  // return left;
};
```

之前我们说过过滤器实际上就只是调用一个函数而已，这里就能看到实际的操作了：我们会生成一个函数调用表达式，函数就是过滤器，而函数的参数就是过滤器左侧的表达式了。

我们还需要在 AST 编译器中做一些额外的工作，但在此之前，我们先得把 `filter` 加入到到 AST 构建器的调用链中。这样我们也就知道了 `filter` 回退到 `assignment`，`filter ` 的优先级比 `assignment` 还要低。事实上，过滤器表达式是优先级最低的表达式。因此我们在处理一个表达式语句时，首先要调用的是 `filter` 方法：

_src/parse.js_

```js
AST.prototype.program = function() {
  // var body = [];
  // while (true) {
  //   if (this.tokens.length) {
      body.push(this.filter());
  //   }
  //   if (!this.expect(';')) {
  //     return { type: AST.Program, body: body };
  //   }
  // }
};
```

当我们使用括号改变优先级时，我们最先要处理的也是过滤器：

_src/parse.js_

```js
AST.prototype.primary = function() {
  // var primary;
  // if (this.expect('(')) {
    primary = this.filter();
  //   this.consume(')');
  // } else if (this.expect('[')) {
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
  // ...
};
```

在 AST 编译器能对 `CallExpression` 做一些有用的事情之前，它需要先知道这个 `CallExpression` 是不是一个过滤器 `CallExpression`。这是因为在普通的调用函数表达式中，这个函数或者方法（也即是 callee）需要在 Scope 的上下文中调用，而 fitler 不需要。我们需要新增一个布尔类型标识 `filter` 供编译器辨识：

_src/parse.js_

```js
AST.prototype.filter = function() {
  // var left = this.assignment();
  // if (this.expect('|')) {
  //   left = {
  //     type: AST.CallExpression,
  //     callee: this.identifier(),
  //     arguments: [left],
      filter: true
  //   };
  // }
  // return left;
};
```

现在在编译器中，我们就可以加一个 `if` 代码块来区分处理 filter 的函数调用：

_src/parse.js_

```js
case AST.CallExpression:
  var callContext, callee, args;
  if (ast.filter) {
    
  } else {
    callContext = {};
    callee = this.recurse(ast.callee, callContext);
    args = _.map(ast.arguments, _.bind(function(arg) {
      // return 'ensureSafeObject(' + this.recurse(arg) + ')';
    }, this));
    // if (callContext.name) {
    //   this.addEnsureSafeObject(callContext.context);
    //   if (callContext.computed) {
    //     callee = this.computedMember(callContext.context, callContext.name);
    //   } else {
    //     callee = this.nonComputedMember(callContext.context, callContext.name);
    //   }
    // }
    // this.addEnsureSafeFunction(callee);
    // return callee + '&&ensureSafeObject(' + callee + '(' + args.join(',') + '))';
  }
  break;
```

我们现在要做的就是加入一个辅助函数 `filter` 来获得过滤器要处理的 JavaScript 代码。然后我们可以使用 `recurse` 来处理传入的参数，然后返回结合了函数和参数的一段代码：

```js
case AST.CallExpression:
  // var callContext, callee, args;
  // if (ast.filter) {
    callee = this.filter(ast.callee.name);
    args = _.map(ast.arguments, _.bind(function(arg) {
      return this.recurse(arg);
    }, this));
    return callee + '(' + args + ')';
  // } else {
  //   // ...
  // }
  // break;
```

现在我们希望过滤器函数在运行时能使用某些变量。我们可以生成一个变量然后返回它：

_src/parse.js_

```js
ASTCompiler.prototype.filter = function(name) {
  var filterId = this.nextId();
  return filterId;
};
```

由于现在我们还没有对这个变量赋值，它的值自然就是 `undefined` 了。我们需要把过滤函数赋值给它。具体来说，我们需要生成一段代码，这段代码会从过滤器服务中获取到对应的过滤器，并在运行时将其放入这个变量中。

在此之前，我们需要记录这个表达式用了哪些过滤器。我们可以在编译阶段保存这些信息到一个新的属性中去：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = {
  //   body: [],
  //   nextId: 0,
  //   vars: [],
    filters: {}
  // };
  // ...
};
```

现在在调用 `filter` 时，我们就可以把过滤器的信息放到 state 属性里去了，属性名就是过滤器的名称，属性值就是过滤器：

_src/parse.js_

```js
ASTCompiler.prototype.filter = function(name) {
  // var filterId = this.nextId();
  this.state.filters[name] = filterId;
  // return filterId;
};
```

如果这个过滤器之后被再次使用，只需要重用上一次生成的变量就可以了，不需要再重新生成：

_src/parse.js_

```js
// ASTCompiler.prototype.filter = function(name) {
  if (!this.state.filters.hasOwnProperty(name)) {
    this.state.filters[name] = this.nextId();
  }
  return this.state.filters[name];
// };
```

此时，一旦 AST 递归处理完毕，`state.filters` 中就会包含所有在表达式中用到的过滤器。现在，我们应该要生成给 filter 变量赋值的代码。为此，我们需要在运行时就能访问到 `filter` 服务，然后把它传递给最终生成的函数，就像我们前面为其他几个函数所做的那样：

_src/parse.js_

```js
return new Function(
  // 'ensureSafeMemberName',
  // 'ensureSafeObject',
  // 'ensureSafeFunction',
  // 'ifDefined',
  'filter',
  // fnString)(
  //   ensureSafeMemberName,
  //   ensureSafeObject,
  //   ensureSafeFunction,
  //   ifDefined,
    filter);
```

我们还需要引入 `filter` 服务到 `parse.js` 中：

_src/parse.js_

```js
// 'use strict'

// var _ = require('lodash');
var filter = require('./filter').filter;
```

我们要生成的第一段代码就是查找过滤器。我们会加入一个辅助函数 `filterPrefix`：

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
  var fnString = this.filterPrefix() +
    // 'var fn=function(s,l){' + (this.state.vars.length ?
    //   'var ' + this.state.vars.join(',') + ';' :
    //   ''
    // ) + this.state.body.join('') + '}; return fn;';
  // ...
};
```

如果表达式中没有用过滤器，就会返回一个空字符串：

_src/parse.js_

```js
ASTCompiler.prototype.filterPrefix = function() {
  if (_.isEmpty(this.state.filters)) {
    return '';
  } else {
    
  }
};
```

如果有用到过滤器，这个方法会构建一个初始化变量的集合，然后为他们生成一个 `var` 声明：

_src/parse.js_

```js
ASTCompiler.prototype.filterPrefix = function() {
  // if (_.isEmpty(this.state.filters)) {
  //   return '';
  // } else {
    var parts = [];
    
    return 'var ' + parts.join(',') + ';';
  // }
};
```

表达式中使用过的每一个过滤器，它在 `ASTCompiler.prototype` 生成的变量名称都会被排除。`filter` 和使用 `filter` 服务查找到并初始化的过滤器，现在就能用于最终生成的代码中了：

_src/parse.js_

```js
ASTCompiler.prototype.filterPrefix = function() {
  // if (_.isEmpty(this.state.filters)) {
  //   return '';
  // } else {
    var parts = _.map(this.state.filters, _.bind(function(varName, filterName) {
      return varName + '=' + 'filter(' + this.escape(filterName) + ')';
    }, this));
  //   return 'var ' + parts.join(',') + ';';
  // }
};
```

这里还有一个剩余的问题，当我们使用 `nextId` 生成变量名时，我们也会把这些变量加入到 `vars` 这个状态变量（state variable）中，这也是 `nextId` 要做的事情。这也意味着，这些变量会在表达式函数中进行声明，会覆盖（shadow） 我们刚才创建的过滤器变量。实际上，如果我们有一个这样的表达式：

```js
42 | increment
```

就会生成类似下面的代码：

```js
function(ensureSafeMemberName, ensureSafeObject, ensureSafeFunction,
  ifDefined, filter) {
  var v0 = filter('increment');
  var fn = function(s, l) {
    var v0;
    return v0(42);
  };
  return fn;
}
```

这里的第二个 `var v0` 就是问题所在。我们需要在调用 `nextId` 时传入一个特殊的标识，让它只管生成变量的 id，不需要再进行声明——因为我们已经在 `filterPrefix` 中单独处理了过滤器变量的声明：

_src/parse.js_

```js
ASTCompiler.prototype.filter = function(name) {
  // if (!this.state.filters.hasOwnProperty(name)) {
    this.state.filters[name] = this.nextId(true);
  // }
  // return this.state.filters[name];
};
```

在 `nextId` 中，如果传入的标识参数为假值，我们需要把生成的变量 ID 放到 `state.vars` 中，否则如果是 `filter` 生成的变量就直接跳过这一步：

_src/parse.js_

```js
ASTCompiler.prototype.nextId = function(skip) {
  // var id = 'v' + (this.state.nextId++);
  if (!skip) {
    this.state.vars.push(id);
  }
  // return id;
};
```

现在我们写的测试用例就都能通过了，我们能在表达式中使用过滤器了！
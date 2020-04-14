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

现在在编译器中，我们就可以加一个 `if` 代码块来区分对待 filter 的函数调用：

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
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
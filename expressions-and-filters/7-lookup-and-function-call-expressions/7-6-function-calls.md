### 函数调用
#### Function Calls

在 Angular 表达式中，函数调用与属性查找一样常用：

_test/parse_spec.js_

```js
it('parses a function call', function() {
  var fn = parse('aFunction()');
  expect(fn({aFunction: function() { return 42; }})).toBe(42);
});
```

首先要理解的事，函数调用实际上包含两个事件：

首先你要找到要调用的函数，像上面的 `‘aFunction’`，然后我们需要使用括号来进行调用。查找函数的过程跟其他属性查找并没有区别。毕竟在 JavaScript 中，函数跟其他值没有区别。

这就意味着我们可以利用现有的代码来完成对函数的查找，剩下来的工作就是调用。我们先把括号变成能被 Lexer 弹出的字符 token：

_src/parse.js_

```js
} else if (this.is('[],{}:.()')) {
  this.tokens.push({
    text: this.ch
  });
  this.index++;
```

跟访问属性一样，函数调用会作为一个 primary AST 节点进行处理。在 `AST.primary` 的 `while` 循环中，我们不仅要消耗方括号和点运算符，还要消耗圆括号。当遇到圆括号时，我们会生成一个 `CallExpression` 节点，然后把前一个 primary 表达式作为被调用函数（callee）：

_src/parse.js_

```js
// var next;
while ((next = this.expect('.', '[', '('))) {
  if (next.text === '[') {
    // primary = {
    //   type: AST.MemberExpression,
    //   object: primary,
    //   property: this.primary(),
    //   computed: true
    // };
    // this.consume(']');
  } else if (next.text === '.') {
    // primary = {
    //   type: AST.MemberExpression,
    //   object: primary,
    //   property: this.identifier(),
    //   computed: false
    // };
  } else if (next.text === '(') {
    primary = { type: AST.CallExpression, callee: primary };
    this.consume(')');
  }
}
```

我们还需要定义常量 `CallExpression`：

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
AST.CallExpression = 'CallExpression';
```

现在，我们准备要把函数调用表达式编译成 JavaScript 代码。首先，我们需要对 `callee` 进行递归处理，获得要调用的函数。然后生成调用函数的代码，只有在函数存在时才会调用函数：

_src/parse.js_

```js
case AST.CallExpression:
  var callee = this.recurse(ast.callee);
  return callee + '&&' + callee + '()';
```

当然，大部分函数调用并不会像上面我们看到的那么简单。调用函数的同时，我们经常会传入参数，但目前我们”幼稚“的函数实现对此一无所知。

表达式应该支持处理简单的参数，比如整数：

_test/parse_spec.js_

```js
it('parses a function call with a single number argument', function() {
  var fn = parse('aFunction(42)');
  expect(fn({aFunction: function(n) { return n; }})).toBe(42);
});
```

我们也应该能够处理来源于 scope 的参数：

_test/parse_spec.js_

```js
it('parses a function call with a single identifier argument', function() {
  var fn = parse('aFunction(n)');
  expect(fn({n: 42, aFunction: function(arg) { return arg; }})).toBe(42);
});
```

有些参数本身就是函数调用：

_test/parse_spec.js_

```js
it('parses a function call with a single function call argument', function() {
  var fn = parse('aFunction(argFn())');
  expect(fn({
    argFn: _.constant(42),
    aFunction: function(arg) { return arg; }
  })).toBe(42);
});
```

当然，如果是用逗号分隔的多个参数，参数也可能是以上各种情况的组合：

_test/parse_spec.js_

```js
it('parses a function call with multiple arguments', function() {
  var fn = parse('aFunction(37, n, argFn())');
  expect(fn({
    n: 3,
    argFn: _.constant(2),
    aFunction: function(a1, a2, a3) { return a1 + a2 + a3; }
  })).toBe(42);
});
```

由于加上了上述单元测试，我们也需要在 `parse_spec.js` 中加入 LoDash 了：

_test/parse_spec.js_

```js
// 'use strict';

var _ = require('lodash');
// var parse = require('../src/parse');
```

在 AST 构建器中，我们需要对在括号中的参数进行解析。我们会使用一个叫 `parseArguments` 的新方法来完成这项任务：

_src/parse.js_

```js
} else if (next.text === '(') { 
  primary = {
    // type: AST.CallExpression,
    // callee: primary,
    arguments: this.parseArguments()
  }; 
  // this.consume(')');
```

这个方法会一直收集 primary 表达式，直到出现一个右圆括号，这跟我们处理数组字面量时的方法完全一样，但我们不支持在参数尾部加入逗号：

_src/parse.js_

```js
AST.prototype.parseArguments = function() {
  var args = [];
  if (!this.peek(')')) {
    do {
      args.push(this.primary());
    } while (this.expect(','));
  }
  return args;
};
```

当我们把这些参数表达式编译为 JavaScript 代码时，我们可以对每一个参数进行递归处理，并把处理结果放到一个数组中保存起来：

_src/parse.js_

```js
case AST.CallExpression:
  // var callee = this.recurse(ast.callee);
  var args = _.map(ast.arguments, _.bind(function(arg) {
    return this.recurse(arg);
  }, this));
  // return callee + '&&' + callee + '()';
```

然后，我们可以把这些参数表达式进行连接，然后加入到要生成的函数调用语句中去：

_src/parse.js_

```js
case AST.CallExpression:
  // var callee = this.recurse(ast.callee);
  // var args = _.map(ast.arguments, _.bind(function(arg) {
  //   return this.recurse(arg);
  // }, this));
  return callee + '&&' + callee + '(' + args.join(',') + ')';
```
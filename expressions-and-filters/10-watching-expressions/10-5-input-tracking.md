### 对输入表达式进行跟踪
#### Input Tracking

关于表达式侦听，我们可以做一个优化，也就是_输入表达式跟踪_（input tracking）。它的核心概念就是当一个表达式是由一个或多个_输入表达式_组成（比如 `'a * b'` 就是由 `'a'` 和 `'b'`组成），除非其中一个输入表达式发生了变化，否则无需再重新计算表达式。

举个例子，如果有一个数组字面量，它里面所有的元素都没有发生变化：

_test/scope_spec.js_

```js
it('does not re-evaluate an array if its contents do not change', function() {
  var values = [];

  scope.a = 1;
  scope.b = 2;
  scope.c = 3;
  
  scope.$watch('[a, b, c]', function(value) {
    values.push(value);
  });
  
  scope.$digest();
  expect(values.length).toBe(1);
  expect(values[0]).toEqual([1, 2, 3]);
  
  scope.$digest();
  expect(values.length).toBe(1);
  scope.c = 4;
  
  scope.$digest();
  expect(values.length).toBe(2);
  expect(values[1]).toEqual([1, 2, 4]);
});
```

这里我们会对一个所有元素都不是常量的数组进行侦听。我们总共会进行 3 次 digest。第一次 digest 后我们希望 listener 函数返回的是包含了所有元素的数组。第二次 digest 时我们不会对数组的内容进行修改。第三次 digest 之前，我们会对数组内容进行修改，这时我们希望 listener 会被重新触发。

实际上现在程序会抛出一个 “10 $digest iterations reached” 的异常，因为我们使用的基于引用的侦听器，这个侦听器每次执行时都会生成一个新的数组，这是行不通的。

我们需要在 parser 生成的每个表达式函数中记录它的输入表达式——这个表达式改变时可能会让整个表达式的值都发生改变。我们需要对我们的 AST 编译器进行扩展，让它不仅支持处理完整的表达式，还能够处理一个表达式中包含的多个输入表达式。

在介绍 AST 编译器的变动前，我们先来看看侦听器这边应该怎么改。当我们传入一个表达式，如果它既不是常量，也不是单次绑定，而且有输入表达式，它就肯定会有一个 `inputs` 属性。如果是的话，我们将会使用一个特殊的 `inputsWatchDelegate` 委托进行监听：

_src/parse.js_

```js
function parse(expr) {
  // switch (typeof expr) {
  //   case 'string':
  //     var lexer = new Lexer();
  //     var parser = new Parser(lexer);
  //     var oneTime = false;
  //     if (expr.charAt(0) === ':' && expr.charAt(1) === ':') {
  //       oneTime = true;
  //       expr = expr.substring(2);
  //     }
  //     var parseFn = parser.parse(expr);
  //     if (parseFn.constant) {
  //       parseFn.$$watchDelegate = constantWatchDelegate;
  //     } else if (oneTime) {
  //       parseFn.$$watchDelegate = parseFn.literal ? oneTimeLiteralWatchDelegate :
  //         oneTimeWatchDelegate;
      } else if (parseFn.inputs) {
        parseFn.$$watchDelegate = inputsWatchDelegate;
      }
  //     return parseFn;
  //   case 'function':
  //     return expr;
  //   default:
  //     return _.noop;
  // }
}
```
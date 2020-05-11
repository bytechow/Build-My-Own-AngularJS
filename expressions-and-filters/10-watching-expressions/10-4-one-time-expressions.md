### 单次绑定表达式
#### One-Time Expressions

作为 Angular 开发者，我们经常会遇到有些 watch 只需要赋值一次，后续不会再发生改变的情况。比较常见的就是像下面这样的对象列表：

```xml
<li ng-repeat="user in users">
  {{user.firstName}} {{user.lastName}}
</li>
```

这段代码使用的 `ng-repeat` 指定首先会为 `users` 生成了一个侦听器集合，但对于每个 `user` 来说，它同时也会增加两个 watcher——第一个是姓（first name），第二个是名（last name）。

一般来说，这样的姓名都不会发生改变，因为它们本身就是只读的——没有任何应用逻辑会去修改它。然而，Angular 现在还是必须在每次 digest 时对它们进行脏值检测，因为 digest 并不知道你不需要对它们进行改变。这些无意义的脏值检测可能会在大型应用和低端设备上造成明显的性能影响。

为了解决这个问题，Angular 提供了单次绑定（one-time binding）的特性。当你知道一个监听器在存续期间的值不会改变时，你可以在监听表达式前面加入两个冒号，以便让 Angular 知道：

```xml
<li ng-repeat="user in users">
  {{::user.firstName}} {{::user.lastName}}
</li>
```

单次绑定监听语法是在表达式引擎中实现的，因为我们可以在任何监听表达式中使用：

_test/scope_spec.js_

```js
it('accepts one-time watches', function() {
  var theValue;

  scope.aValue = 42;
  scope.$watch('::aValue', function(newValue, oldValue, scope) {
    theValue = newValue;
  });
  scope.$digest();
  
  expect(theValue).toBe(42);
});
```

单次绑定的监听器与普通的监听器最大的区别在于，当 watcher 完成运算后会自动销毁，这样它就不会再对 digest 循环造成压力了：

_test/scope_spec.js_

```js
it('removes one-time watches after first invocation', function() {
  scope.aValue = 42;
  scope.$watch('::aValue', function() {});
  scope.$digest();
  
  expect(scope.$$watchers.length).toBe(0);
});
```

单次绑定的监听的实现都在 `parse.js` 里，利用上一个节中介绍的监听委托系统。如果表达式前面有两个冒号，我们就把“单次绑定的监听委托”（one-time watch delegate）作为监听委托：

_src/parse.js_

```js
function parse(expr) {
  // switch (typeof expr) {
  //   case 'string':
  //     var lexer = new Lexer();
  //     var parser = new Parser(lexer);
      var oneTime = false;
      if (expr.charAt(0) === ':' && expr.charAt(1) === ':') {
        oneTime = true;
        expr = expr.substring(2);
      }
      // var parseFn = parser.parse(expr);
      // if (parseFn.constant) {
      //   parseFn.$$watchDelegate = constantWatchDelegate;
      } else if (oneTime) {
        parseFn.$$watchDelegate = oneTimeWatchDelegate;
  //     }
  //     return parseFn;
  //   case 'function':
  //     return expr;
  //   default:
  //     return _.noop;
  // }
}
```

从表面上看，单次绑定的行为跟常量侦听委托一模一样：运行一次监听器，然后移除它。确定，如果我们按照常量侦听委托的方法来实现单次绑定监听委托缺失可以通过测试用例：

_src/parse.js_

```js
function oneTimeWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  var unwatch = scope.$watch(function() {
    return watchFn(scope);
  }, function(newValue, oldValue, scope) {
    if (_.isFunction(listenerFn)) {
      listenerFn.apply(this, arguments);
    }
    unwatch();
  }, valueEq);
  return unwatch;
}
```
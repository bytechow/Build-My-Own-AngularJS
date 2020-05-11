### 优化对常量表达式的侦测

#### Optimizing Constant Expression Watching

如果在 watch 中使用的是表达式字符串，我们就可以做一些优化来提升 digest 循环运行速度。上一节我们已经看到如何将常量表达式的 `constant` 属性设置为 `true`。常量表达式返回的是一个固定值。这意味着常量表达式的 watch 被触发执行第一次，以后它都不会再变“脏”（dirty）了。同时也意味着我们可以把这个 watch 用安全的方式移除掉，这样就能减轻之后 digest 进行脏值检测的负担了。下面我们在 `scope_spec.js` 文件的 `describe('$digest')` 测试集中加入这个测试用例：

_test/scope\_spec.js_

```js
it('removes constant watches after first invocation', function() {
  scope.$watch('[1, 2, 3]', function() {});
  scope.$digest();

  expect(scope.$$watchers.length).toBe(0);
});
```

现在，这个测试用例执行后会抛出一个“10 iterations reached”（已重试了 10 次）的异常，因为这个表达式每次执行都会生成一个新的数组，因此使用引用比较的 watch 每次都会认为它是一个新值。但 `[1, 2, 3]` 实际上是一个常量，我们只需要执行一次运算就够了。

我们可以使用一个叫 _侦测委托_（watch delegates） 的新表达式特性来解决这个问题。侦测委托实际上是一个可以附加到表达式上的函数。当 `Scope.$watch` 中发现了表达式带有一个侦测委托，我们就要绕过正常的创建侦听器的步骤。我们不会在这个方法中创建侦听器，但是我们会把这个任务委托给表达式本身：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  // var self = this;

  watchFn = parse(watchFn);

  if (watchFn.$$watchDelegate) {
    return watchFn.$$watchDelegate(self, listenerFn, valueEq, watchFn);
  }
  
  // var watcher = {
    watchFn: watchFn,
  //   listenerFn: listenerFn || function() {},
  //   last: initWatchVal,
  //   valueEq: !!valueEq
  // };
  // this.$$watchers.unshift(watcher);
  // this.$$root.$$lastDirtyWatch = null;
  // return function() {
  //   var index = self.$$watchers.indexOf(watcher);
  //   if (index >= 0) {
  //     self.$$watchers.splice(index, 1);
  //     self.$$root.$$lastDirtyWatch = null;
  //   }
  // };
};
```

为了正确创建侦听器，我们需要传递所有必须的参数给侦听委托：作用域实例、listener 函数，基于值/引用的相等性判断标识还有要侦听的表达式本身。

> 可以看到，侦听委托加入了双美元符号作为前缀，这表示它是 Angular 内部使用的特性，应用开发者不应该直接使用。

只要我们需要在侦听器针对某些特殊情况进行处理，表达式解析器都可以把侦听委托加入到表达式函数上了。其中一个特殊情况就是常量表达式，我们需要加入一个_常量侦听委托_（constant watch delegate）：

_src/parse.js_

```js
function parse(expr) {
  // switch (typeof expr) {
  //   case 'string':
  //     var lexer = new Lexer();
  //     var parser = new Parser(lexer);
      var parseFn = parser.parse(expr);
      if (parseFn.constant) {
        parseFn.$$watchDelegate = constantWatchDelegate;
      }
      return parseFn;
  //   case 'function':
  //     return expr;
  //   default:
  //     return _.noop;
  // }
}
```

常量侦听委托（constant watch delegate）跟其他 watcher 基本一致，但它会在第一次调用后移除自身：

_src/parse.js_

```js
function constantWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  var unwatch = scope.$watch(
    function() {
      return watchFn(scope);
    },
    function(newValue, oldValue, scope) {
      if (_.isFunction(listenerFn)) {
        listenerFn.apply(this, arguments);
      }
      unwatch();
    },
    valueEq);
  return unwatch;
}
```

注意，我们并不是直接把原始的 `watchFn` 作为 `$watch` 的第一个参数，因为这样会让 `$watch` 再次找到 `$$watchDelegate`，形成无限循环。我们会把它包裹在一个不带 `$$watchDelegate` 的函数中。

同时也要注意我们会返回 `unwatch` 函数。即使常量侦听器会在第一次调用后销毁自身，我们也需要允许它使用 `Scope.$watch` 的返回值来执行移除操作，跟其他侦听器保持一致。
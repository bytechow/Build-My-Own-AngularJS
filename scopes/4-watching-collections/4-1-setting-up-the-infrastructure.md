### 建立基础设施
#### Setting Up The Infrastructure

我们先为 `Scope` 上的 `$watchCollection` 创建一个代码空间。

这个函数的签名跟 `$watch` 很像：它会接收一个会返回我们要侦听的值的 watch 函数，还有一个会在侦听到变化时调用的 listener 函数。在内部，函数会将侦听任务委托给 `$watch`，并使用本地创建的 watch 函数和 listener 函数作为参数：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {

  var internalWatchFn = function(scope) {
  };
  
  var internalListenerFn = function() {

  };
  
  return this.$watch(internalWatchFn, internalListenerFn);
};
```

你可能还记得，`$watch` 函数会返回一个用于移除对应 watcher 的函数。我们也通过将该函数返回给原始调用者来实现移除 `$watchCollection` 注册的 watcher 的功能。

跟上一章类似，我们也为这个功能添加一个 `describe` 代码块。它应该作为 `Scope` 下的顶层 `describe` 代码块。

_test/scope_spec.js_

```js
describe('$watchCollection', function() {

  var scope;
  
  beforeEach(function() {
    scope = new Scope();
  });

});
```
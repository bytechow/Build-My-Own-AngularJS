### 建立基础设施

#### Setting Up The Infrastructure

我们先为 `Scope` 上的 `$watchCollection` 创建一个代码空间。

这个函数的签名跟 `$watch` 很像：它接收一个返回侦听值的 watch 函数，还有一个会在侦听到变化时调用的 listener 函数。实际上，在函数内部会将侦听任务委托给 `$watch`，调用 `$watch` 时传入的 watch 和 listener 都是在函数内部创建的：

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

如果你还记得的话，`$watch` 会返回一个函数，用于移除当前 watcher。这里我们直接返回这个函数，这就实现移除 `$watchCollection` 的功能了。

跟上一章一样，我们会在 `Scope` 测试文件的顶层新建一个 `describe` 代码块，专门用于存放与这个功能相关的测试用例。

_test/scope\_spec.js_

```js
describe('$watchCollection', function() {

  var scope;

  beforeEach(function() {
    scope = new Scope();
  });

});
```




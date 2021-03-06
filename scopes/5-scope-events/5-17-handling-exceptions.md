### 异常处理
#### Handling Exceptions

接下来，我们还剩一件事情还没有处理，那就是对一些异常情况进行处理。当一个 listener 函数中发生了异常，我们希望事件会继续被传播。而我们的代码要做到的不仅是如此，我们还要让异常一直向上抛出直到 `$emit` 或 `$broadcast` 的调用者那里。这意味着下面这些测试（在同时测试 `$emit` 和 `$broadcast` 的 `forEach` 代码块中加入）能抛出异常：

_test/scope_spec.js_

```js
it('does not stop on exceptions on '+method, function() {
  var listener1 = function(event) {
    throw 'listener1 throwing an exception';
  };
  var listener2 = jasmine.createSpy();
  scope.$on('someEvent', listener1);
  scope.$on('someEvent', listener2);

  scope[method]('someEvent');
  
  expect(listener2).toHaveBeenCalled();
});
```

与 watch 函数、`$evalAsync` 函数和 `$$postDigest` 函数一样，我们需要在调用 listener 函数的代码外层包裹 `try...catch` 代码块，然后处理异常。现在我们的处理很简单，就只是把错误打印到控制台而已，但以后，我们会把错误传递到一个特殊的异常处理服务中进行处理：

```js
Scope.prototype.$$fireEventOnScope = function(eventName, listenerArgs) {
  // var listeners = this.$$listeners[eventName] || [];
  // var i = 0;
  while (i < listeners.length) {
    // if (listeners[i] === null) {
    //   listeners.splice(i, 1);
    // } else {
      try {
        // listeners[i].apply(null, listenerArgs);
      } catch (e) {
        console.error(e);
      }
      // i++;
    // }
  }
};
```
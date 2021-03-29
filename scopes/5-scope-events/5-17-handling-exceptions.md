### 异常处理
#### Handling Exceptions

我们还漏了一件事情没做，那就是异常处理。如果在执行 listener 函数的过程中出现了异常，我们希望事件能继续传播下去。除此之外，我们还要让异常一直向上抛出直至到达 `$emit` 或 `$broadcast` 的调用者。这意味着下面这个单元测试能抛出异常（在同时测试 `$emit` 和 `$broadcast` 的 `forEach` 代码块加入这个单元测试即可）：

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

与 watch 、`$evalAsync` 和 `$$postDigest` 三个函数一样，我们需要在调用 listener 函数的代码外围包裹 `try...catch` 代码块，从而实现异常的处理。现在我们的处理方法很简单，只是把错误打印到控制台而已，之后我们会把错误传递到一个特定的异常处理服务中进行处理：

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
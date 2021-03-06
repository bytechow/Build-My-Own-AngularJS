### 停止事件传播
#### Stopping Event Propagation

DOM 事件还有另一个非常常用的特性，就是可以停止事件继续传播。DOM 事件对象中包含了一个叫 `stopPropagation` 的方法来完成这个功能。它能够完成类似这样的功能：多级的 DOM 中都注册了点击事件监听器，当其中一级的 DOM 上触发了点击事件时，我们不希望在所有层级上的 DOM 都接收、处理这个事件。

作用域事件也会有一个 `stopPropagation` 的方法，但只会出现在通过发出（`emit`）的方式进行传播的事件上。而通过广播（`broadcast`）传播的事件是无法被停止的。（再次说明了广播事件成本的昂贵）。

这就意味着，当你 emit 一个事件的时候，且其中一个 listener 停止事件冒泡，在后面的父作用域事件就不会再接收到这个事件了：

_test/scope_spec.js_

```js
it('does not propagate to parents when stopped', function() {
  var scopeListener = function(event) {
    event.stopPropagation();
  };
  var parentListener = jasmine.createSpy();

  scope.$on('someEvent', scopeListener);
  parent.$on('someEvent', parentListener);
  
  scope.$emit('someEvent');
  
  expect(parentListener).not.toHaveBeenCalled();
});
```

这个事件虽然不会传递到父作用域去，但特别要注意的是，这个事件依然会传递给当前作用域的其他 listener，只是停止向父作用域传递而已：

_test/scope_spec.js_

```js
it('is received by listeners on current scope after being stopped', function() {
  var listener1 = function(event) {
    event.stopPropagation();
  };
  var listener2 = jasmine.createSpy();

  scope.$on('someEvent', listener1);
  scope.$on('someEvent', listener2);
  
  scope.$emit('someEvent');
  
  expect(listener2).toHaveBeenCalled();
});
```

首先我们需要做的是设置一个布尔值标识，用于识别当前是否已经在某处调用了 `stopPropagation`。我们可以在 `$emit` 利用闭包来引入这个标识。然后，我们肯定需要在事件对象中加入 `stopPropagation` 函数。最后，在 `$emit` 函数的 `do...while` 循环进入上一层作用域之前，我们要对这个状态进行检查：

```js
Scope.prototype.$emit = function(eventName) {
  var propagationStopped = false;
  var event = {
    // name: eventName,
    // targetScope: this,
    stopPropagation: function() {
      propagationStopped = true;
    }
  };
  // var listenerArgs = [event].concat(_.tail(arguments));
  // var scope = this;
  do {
    // event.currentScope = scope;
    // scope.$$fireEventOnScope(eventName, listenerArgs);
    // scope = scope.$parent;
  } while (scope && !propagationStopped);
  return event;
};
```
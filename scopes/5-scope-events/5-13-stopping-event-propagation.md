### 停止事件传播
#### Stopping Event Propagation

DOM 事件还有另一个非常常用的特性，就是可以停止事件继续传播。DOM 事件对象中有一个叫 `stopPropagation` 的方法来完成这个功能。它的使用场景类似这样：DOM 树上有多个层次的元素注册了点击事件监听器，当其中一个 DOM 元素触发点击事件时，我们不希望调用其他层级上的事件监听器。

作用域事件也有 `stopPropagation` 方法，但只有发出（`emit`）这种事件传播方式才有。通过广播（`broadcast`）传播的事件是无法停止的。（再次说明了广播事件成本之高昂）。

这意味着，当你发出一个事件，而该事件其中一个监听器停止了事件的传播，那这在之后的父作用域就不会再接收到这个事件了：

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

但要特别注意的是，这个事件虽然不会传递到父作用域去，但依然会传递给当前作用域的其他 listener：

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

我们要设置一个布尔值变量，用于识别当前是否已经在某处调用了 `stopPropagation`。我们可以在 `$emit` 的闭包中引入这个标识。接着，我们要在事件对象中加入 `stopPropagation` 函数。最后，在 `$emit` 函数的 `do...while` 遍历上一层作用域之前，我们要先对这个布尔值变量的状态进行检查：

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
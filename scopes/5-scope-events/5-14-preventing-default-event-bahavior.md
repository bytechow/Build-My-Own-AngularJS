### 阻止事件的默认行为
#### Preventing Default Event Behavior

除了 `stopPropagation`，DOM 事件还有另一种取消方式，那就是阻止它们的“默认行为”，也就是 DOM 事件中的 `preventDefault` 函数。它的目的主要在于防止浏览器中某些事件带有的副作用，但依然允许这个事件的 所有 listener 能得到通知。举例来说，当我们在超链接元素的点击事件处理函数上调用 `preventDefault`时，那当这个超链接被点击时，浏览器不会发生跳转，但相关的所有点击事件处理函数依然会被调用。

作用域事件也有 `preventDefault` 函数，而且无论事件是以 emit 还是 broadcast 的形式发出的，都会提供这个函数。但因为作用域事件并没有什么内建的“默认行为”，调用这个函数并没有什么意义。它只会做一件事，就是在事件对象中设置一个布尔值标识 `defaultPrevented`。这个标识并不会改变作用域事件的行为，但也有可能用在自定义事件结束传播时决定是否要触发某些默认行为。Angular 的 `$locationService` 在广播 location 事件时就用到了这个函数。

因此，我们要做的就是添加一个测试，在这个测试中验证当在事件对象上调用了 `preventDefault()` 之后，时间对象中的 `defaultPrevented` 标识是否被设置成功了。由于 `$emit` 和 `$broadcast` 上都有这个特性，我们会把下面的这个测试放到之前处理两种事件传播方式的公共行为的循环中：

_test/scope_spec.js_

```js
it('is sets defaultPrevented when preventDefault called on '+method, function() {
  var listener = function(event) {
    event.preventDefault();
  };
  scope.$on('someEvent', listener);

  var event = scope[method]('someEvent');
  
  expect(event.defaultPrevented).toBe(true);
});
```

实现起来也跟 `stopPropagation` 大同小异：新增一个函数，这个函数会在事件对象上添加一个布尔值标识。但区别在于这次我们是直接把布尔值标识作为事件对象的属性进行赋值，也不再需要根据这个布尔值标识来做什么判断了。下面是 `$emit` 的实现方法：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  // var propagationStopped = false;
  var event = {
    // name: eventName,
    // targetScope: this,
    // stopPropagation: function() {
    //   propagationStopped = true;
    // },
    preventDefault: function() {
      event.defaultPrevented = true;
    }
  };
  // var listenerArgs = [event].concat(_.tail(arguments));
  // var scope = this;
  // do {
  //   event.currentScope = scope;
  //   scope.$$fireEventOnScope(eventName, listenerArgs);
  //   scope = scope.$parent;
  // } while (scope && !propagationStopped);
  // return event;
};
```

在 `$broadcast` 中也是一样的：

_src/scope.js_

```js
Scope.prototype.$broadcast = function(eventName) {
  var event = {
    // name: eventName,
    // targetScope: this,
    preventDefault: function() {
      event.defaultPrevented = true;
    }
  };
  // var listenerArgs = [event].concat(_.tail(arguments));
  // this.$$everyScope(function(scope) {
  //   event.currentScope = scope;
  //   scope.$$fireEventOnScope(eventName, listenerArgs);
  //   return true;
  // });
  // return event;
};
```
### 阻止事件的默认行为
#### Preventing Default Event Behavior

除了 `stopPropagation` 之外，我们还可以通过 `preventDefault` 方法“取消” DOM 事件，也就是阻止事件在浏览器的“默认行为”。虽然如此，在调用这个方法后，这个事件的所有监听器依旧会被调用。举例来说，当我们在超链接元素的点击事件监听函数上调用 `preventDefault`后，浏览器不会跟踪该超链接（也就是不会跳转到超链接对应的地址），但相关的点击事件处理函数依然会被调用。

作用域事件也有自己的 `preventDefault` 方法，而且无论事件是以 emit 还是 broadcast 的形式进行传播的都有这个方法。但因为作用域事件并没有什么内建的“默认行为”，调用这个函数并没有什么意义。它只会在事件对象中设置一个布尔值标识 `defaultPrevented`。这个标识并不会改变作用域事件系统的行为，但也有可能用在某些情境下，比如自定义指令，用于决定在事件结束传播时是否要触发某些默认行为。Angular 的 `$locationService` 在广播 location 事件时就用到了这个函数。

因此我们要增加一个测试，验证在事件对象上调用了 `preventDefault()` 之后，事件对象中的 `defaultPrevented` 标识发生了变化，这种行为对于 `$emit` 和 `$broadcast` 来说都是一样。我们把下面这个用例放到之前处理公共行为的循环就可以了：

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

实现 `preventDefault` 的方法也跟 `stopPropagation` 大同小异：调用这个函数后会对一个布尔值变量进行赋值。但区别在于这次的布尔值变量是事件对象上的一个属性，我们也不用再加入与这个布尔值变量相关的判断了。下面是 `$emit` 实现 `preventDefault` 方法：

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
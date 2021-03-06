### 注销事件监听器
#### Deregistering Event Listeners

在分析 `$emit` 和 `$broadcast` 的不同之处之前，我们需要解决一个非常常见的需求：我们不仅可以注册事件监听器，还要能注销它们。

注销事件监听器的方法跟我们在第一章中实现的注销 watcher 的方法其实是一模一样的：注册函数本身就会返回一个注销函数。一旦调用注销函数，listener 就不会再接收到任何事件了：

_test/scope_spec.js_

```js
it('can be deregistered ' + method, function() {
  var listener = jasmine.createSpy();
  var deregister = scope.$on('someEvent', listener);
  
  deregister();
  
  scope[method]('someEvent');
  
  expect(listener).not.toHaveBeenCalled();
});
```

移除监听器最简单的方式，跟我们对 watcher 的处理也是完全相同的——只是通过 splice 的方式将 listener 从集合中提取出来而已：

_src/scope.js_

```js
Scope.prototype.$on = function(eventName, listener) {
  // var listeners = this.$$listeners[eventName];
  // if (!listeners) {
  //   this.$$listeners[eventName] = listeners = [];
  // }
  // listeners.push(listener);
  return function() {
    var index = listeners.indexOf(listener);
    if (index >= 0) {
      listeners.splice(index, 1);
    }
  };
};
```

但是，我们还需要处理一种特殊的情况。在 listener 调用的同时移除 listener 自身的情况是很常见的，例如，我们只希望这个 listener 被调用一次而已。当在我们遍历 listener 数组时出现了这种移除，结果就是这次遍历会跳过一个 listener——也就是紧接着被删除的 listener 的下一个：

_test/scope_spec.js_

```js
it('does not skip the next listener when removed on '+method, function() {
  var deregister;

  var listener = function() {
    deregister();
  };
  var nextListener = jasmine.createSpy();
  
  deregister = scope.$on('someEvent', listener);
  scope.$on('someEvent', nextListener);
  
  scope[method]('someEvent');
  
  expect(nextListener).toHaveBeenCalled();
});
```

这就意味着，我们不能直接移除这个 listener。我们要做的是用一个能表示 listener 已经被删除的特定值来替换这个 listener。`null` 就可以很好地满足这个目的了：

_src/scope.js_

```js
Scope.prototype.$on = function(eventName, listener) {
  // var listeners = this.$$listeners[eventName];
  // if (!listeners) {
  //   this.$$listeners[eventName] = listeners = [];
  // }
  // listeners.push(listener);
  // return function() {
  //   var index = listeners.indexOf(listener);
  //   if (index >= 0) {
      listeners[index] = null;
  //   }
  // };
};
```

然后，在遍历 listener 的时候，我们可以检查一下这个 listener 是否已经变成 `null` 值了，然后再进行删除。另外，我们确实还需要将 `_.forEach` 换成一个手动编写的 `while` 循环来实现这个功能：

_src/scope.js_

```js
Scope.prototype.$$fireEventOnScope = function(eventName, additionalArgs) {
  // var event = { name: eventName };
  // var listenerArgs = [event].concat(additionalArgs);
  // var listeners = this.$$listeners[eventName] || [];
  var i = 0;
  while (i < listeners.length) {
    if (listeners[i] === null) {
      listeners.splice(i, 1);
    } else {
      listeners[i].apply(null, listenerArgs);
      i++;
    }
  }
  // return event;
};
````
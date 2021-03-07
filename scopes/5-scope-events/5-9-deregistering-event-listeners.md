### 注销事件监听器
#### Deregistering Event Listeners

在分析 `$emit` 和 `$broadcast` 的不同之处之前，我们还需要解决一个非常常见的需求：我们不仅要能注册事件监听器，还要能注销它们。

注销事件监听器的方法跟我们在第一章注销 watcher 的方法其实是一模一样的：注册函数本身会返回一个注销函数。一旦调用注销函数，listener 就不会再接收到任何事件通知了：

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

简易版的移除功能跟处理 watcher 的方式完全相同——就是利用 splice 方法将 listener 从集合中提取出来而已：

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

但我们还需要处理一种特殊情况——在调用 listener 的同时移除掉这个 listener。这种情况还是很常见的，比如我们在某些场景下只希望 listener 被调用一次。若在我们遍历 listener 数组时发生了这种移除，会导致这次遍历会跳过一个 listener——紧接着被删除 listener 的那个：

_test/scope_spec.js_

```js
it('does not skip the next listener when removed on '+ method, function() {
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

这意味着我们不能直接移除这个 listener。我们要做的是用一个能表示 listener 已经被删除的特定值来替换这个 listener。`null` 作为这个特定值就很合适了：

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

然后在遍历 listener 时，我们可以检查一下这个 listener 是否已经变成 `null` 值，再决定要执行什么操作。另外，我们确实还需要将 `_.forEach` 换成一个手动编写的 `while` 循环来实现这个功能：

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

> 译者注：注意，这里换成 `while` 循环是因为它可以控制循环变量是否自增。这里一旦遇到有事件需要被移除，就不会递增循环变量。这样即使在删除元素后，循环依然能对被删除元素的后一个元素进行处理。
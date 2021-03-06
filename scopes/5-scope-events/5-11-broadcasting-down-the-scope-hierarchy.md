### 往下层作用域广播事件
#### Broadcasting Down The Scope Hierarchy

`$broadcast` 基本上就是 `$emit` 的“镜像反射”：它会调用当前作用域及其后代作用域上的 listener——无论这个作用域是不是孤立（isolated）的：

_test/scope_spec.js_

```js
it('propagates down the scope hierarchy on $broadcast', function() {
  var scopeListener = jasmine.createSpy();
  var childListener = jasmine.createSpy();
  var isolatedChildListener = jasmine.createSpy();

  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  isolatedChild.$on('someEvent', isolatedChildListener);

  scope.$broadcast('someEvent');
  
  expect(scopeListener).toHaveBeenCalled();
  expect(childListener).toHaveBeenCalled();
  expect(isolatedChildListener).toHaveBeenCalled();
});
```

为了完整起见，我们要确保 `$broadcast` 也会将同一个事件对象传递给所有的 listener：

_test/scope_spec.js_

```js
it('propagates the same event down on $broadcast', function() {
  var scopeListener = jasmine.createSpy();
  var childListener = jasmine.createSpy();

  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  
  scope.$broadcast('someEvent');
  
  var scopeEvent = scopeListener.calls.mostRecent().args[0];
  var childEvent = childListener.calls.mostRecent().args[0];
  expect(scopeEvent).toBe(childEvent);
});
```

在 `$broadcast` 中遍历作用域可就不像在 `$emit` 那么简单直接了，毕竟通往下层作用域并不是一条单一的线路。作用域会分岔形成一个树结构，所以我们需要进行树遍历。更准确地来说，我们需要类似 `$$digestOnce` 中用到的深度优先遍历。实际上，我们可以直接利用第 2 章实现的 `$$everyScope` 函数来完成这个功能：

_src/scope.js_

```js
Scope.prototype.$broadcast = function(eventName) {
  // var event = { name: eventName };
  // var listenerArgs = [event].concat(_.tail(arguments));
  this.$$everyScope(function(scope) {
    scope.$$fireEventOnScope(eventName, listenerArgs);
    return true;
  });
  // return event;
};
```

现在，我们就能够清楚地看到广播（broadcast）事件为什么比发出（emit）事件开销大得多了。往上层作用域发出事件只会有一条路径，而一般作用域层次不会太深。而另一方面，广播事件需要遍历当前的作用域树。也就是说，如果你在根作用域广播一个事件，那么程序会访问整个应用程序中的每一个作用域。
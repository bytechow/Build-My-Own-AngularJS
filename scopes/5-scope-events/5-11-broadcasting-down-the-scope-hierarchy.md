### 往下层作用域广播事件
#### Broadcasting Down The Scope Hierarchy

`$broadcast` 基本上就是 `$emit` 的“镜像”：它会调用当前作用域及其后代作用域上的 listener——无论这个作用域是不是隔离（isolated）的：

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

为了保持一致性，我们也要确保 `$broadcast` 会将同一个事件对象传递给所有的 listener：

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

`$broadcast` 遍历作用域可就不像 `$emit` 那么直接了，毕竟通往底层作用域可不止一条线路。下层作用域会分叉形成一个树结构。我们需要对作用域树进行遍历，确切来说，我们需要类似 `$$digestOnce` 中用到的深度优先遍历算法。实际上，我们可以重用第 2 章实现的 `$$everyScope` 函数来完成这个功能：

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

现在就能清晰地看到广播（broadcast）事件的开销为什么要比发出（emit）事件要大得多了。往上层作用域发出事件有且仅有一条路径，而且作用域层次一般都不会太深。而广播事件则需要对当前以下整个作用域树进行遍历。也就是说，如果你在根作用域广播一个事件，那么程序将会遍历整个应用程序中的每一个作用域。
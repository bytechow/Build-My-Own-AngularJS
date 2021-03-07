### 往上层作用域发出事件
#### Emitting Up The Scope Hierarchy

现在我们终于可以介绍 `$emit` 和 `$broadcast` 的不同之处了：事件在树结构中传播方向不同。

当你 emit 一个事件时，这个事件会传递给当前作用域上的 listener，然后一路往上直至根作用域都会接收到这个事件。

我们需要下面这个单元测试放到我们早前创建的 `forEach` 循环之外，因为它只跟 `$emit` 有关：

_test/scope_spec.js_

```js
it('propagates up the scope hierarchy on $emit', function() {
  var parentListener = jasmine.createSpy();
  var scopeListener = jasmine.createSpy();

  parent.$on('someEvent', parentListener);
  scope.$on('someEvent', scopeListener);

  scope.$emit('someEvent');
  
  expect(scopeListener).toHaveBeenCalled();
  expect(parentListener).toHaveBeenCalled();
});
```

我们先来试着在 `$emit` 中通过向上查找的方法，尽可能简单地实现这个功能。我们可以使用在第 2 章实现的 `$parent` 属性来递归访问作用域的父节点：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  // var additionalArgs = _.tail(arguments);
  var scope = this;
  do {
    scope.$$fireEventOnScope(eventName, additionalArgs);
    scope = scope.$parent;
  } while (scope);
};
```

这近乎于完成功能了。我们现在已经破坏了将事件对象返回给 `$emit` 调用者的约定，还记得我们说过把同一个事件对象传递给每一个 listener 是很重要的吗？即使跨越了不同的作用域也应该要遵守这个规则，但我们并没有通过下面这个测试：

_test/scope_spec.js_

```js
it('propagates the same event up on $emit', function() {
  var parentListener = jasmine.createSpy();
  var scopeListener = jasmine.createSpy();

  parent.$on('someEvent', parentListener);
  scope.$on('someEvent', scopeListener);
  
  scope.$emit('someEvent');
  
  var scopeEvent = scopeListener.calls.mostRecent().args[0];
  var parentEvent = parentListener.calls.mostRecent().args[0];
  expect(scopeEvent).toBe(parentEvent);
});
```

这意味着我们需要撤销之前所做的一些为了减少重复的操作，分别为 `$emit` 和 `$broadcast` 构建各自的事件对象，然后再把它传递给 `$$fireEventOnScope`。要实现这样的效果，我们需要把构建 `listenerArgs` 的过程从 `$$fireEventOnScope` 中抽离出来，这样我们就不用再在每个递归到的 Scope 里再构建它了：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  var event = { name: eventName };
  var listenerArgs = [event].concat(_.tail(arguments));
  // var scope = this;
  do {
    scope.$$fireEventOnScope(eventName, listenerArgs);
    // scope = scope.$parent;
  } while (scope);
  return event;
};

Scope.prototype.$broadcast = function(eventName) {
  var event = { name: eventName };
  var listenerArgs = [event].concat(_.tail(arguments));
  this.$$fireEventOnScope(eventName, listenerArgs);
  return event;
};

Scope.prototype.$$fireEventOnScope = function(eventName, listenerArgs) {
  // var listeners = this.$$listeners[eventName] || [];
  // var i = 0;
  // while (i < listeners.length) {
  //   if (listeners[i] === null) {
  //     listeners.splice(i, 1);
  //   } else {
  //     listeners[i].apply(null, listenerArgs);
  //     i++;
  //   }
  // }
};
``` 

现在还是会有重复代码，但并无大碍。随着 `$emit` 和 `$broadcast` 开始出现更大的分歧，这种重复反而能帮上忙。
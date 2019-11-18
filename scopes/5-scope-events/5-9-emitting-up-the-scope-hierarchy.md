### 往上层作用域发出事件
#### Emitting Up The Scope Hierarchy

终于可以介绍 `$emit` 和 `$broadcast` 的区别：事件在树结构中传播方向不同。

当你 emit 一个事件时，这个事件对象会在当前作用域上传递给 listener，然后一路往上传递直至根作用域，根作用域也会接收到这个事件。

我们需要把对应的单元测试放到我们早前创建的 `forEach` 循环之外，因为它只跟 `$emit` 有关：

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

我们先来试着在 `$emit` 中通过向上查找的方法，尽可能简单地实现这个功能。我们可以使用在本书第 2 章实现的 `$parent` 属性来访问作用域的每一个父节点：

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

这几乎奏效了。
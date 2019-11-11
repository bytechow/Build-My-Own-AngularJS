### 注册事件监听器：$on
#### Registering Event Listeners: $on

要在事件触发时得到通知，你需要先注册这个事件。在 AngularJS 中是通过调用 `$on` 来注册事件的。这个函数会接收两个参数：感兴趣的事件名称，还有需要在事件触发时调用的回调函数。

> AngularJS 中的订阅者被称为 _listener_（监听器），下面我们会继续使用这个名词。

通过 `$on` 注册的监听器既能接收 emit 发出的事件，也能接收 broadcast 广播的事件。实际上，除了事件名称，没有任何方法可以限制其他任何要接收到的事件。

`$on` 函数实际上做了什么呢？它要在某处保存监听器，以便之后事件被触发时能够查找到这个监听器。我们会使用一个对象属性 `$$listeners` 来存储（以两个美元符号作为前缀说明这个属性被认为是私有的）。对象里的 key 就是事件名称，而属性值，而属性值就是一个存放了注册了特定事件的监听器数组。因此，我们可以先测试一下 `$on` 注册的监听器是否被依次保存了：

_test/scope_spec.js_

```js
it('allows registering listeners', function() {
  var listener1 = function() { };
  var listener2 = function() { };
  var listener3 = function() { };

  scope.$on('someEvent', listener1);
  scope.$on('someEvent', listener2);
  scope.$on('someOtherEvent', listener3);
  
  expect(scope.$$listeners).toEqual({
    someEvent: [listener1, listener2],
    someOtherEvent: [listener3]
  });

});
```

我们希望在作用域上存在 `$$listeners` 对象，所以需要先在构造函数上设置好：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  // this.$$applyAsyncQueue = [];
  // this.$$applyAsyncId = null;
  // this.$$postDigestQueue = [];
  // this.$root = this;
  // this.$$children = [];
  this.$$listeners = {};
  // this.$$phase = null;
}
```

在 `$on` 函数内，我们要检查给定的事件是否已经存在对应的监听器数组了，如果还没有，我们需要初始化一个。初始化之后，我们就可以直接把新注册的的监听器放到这个集合中来：

_src/scope.js_

```js
Scope.prototype.$on = function(eventName, listener) {
  var listeners = this.$$listeners[eventName];
  if (!listeners) {
    this.$$listeners[eventName] = listeners = [];
  }
  listeners.push(listener);
};
```

由于侦听器在 Scope 树结构中的位置很重要，目前的 `$$listeners` 还有一个小问题：所有树结构上的监听器都放到同一个 `$$listeners` 集合中去。我们需要的是为每一个作用域建立一个独立的 `$$listeners` 集合。

_test/scope_spec.js_

```js
it('registers different listeners for every scope', function() {
  var listener1 = function() { };
  var listener2 = function() { };
  var listener3 = function() { };

  scope.$on('someEvent', listener1);
  child.$on('someEvent', listener2);
  isolatedChild.$on('someEvent', listener3);
  
  expect(scope.$$listeners).toEqual({someEvent: [listener1]});
  expect(child.$$listeners).toEqual({someEvent: [listener2]});
  expect(isolatedChild.$$listeners).toEqual({someEvent: [listener3]});
});
```

这个单元测试失败了，因为 `scope` 和 `child` 是可以访问到 `$$listeners` 集合的，但 `isolatedChild` 并不能访问这个集合。我们需要对子作用域的构造函数进行改造，让它们显式地管理自己的的 `$$listeners` 集合。对于非孤立的作用域来说，会屏蔽父作用域上的同名属性。这也是我们在第二章中对 `$$watchers` 使用过的解决方案：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated, parent) {
  // var child;
  // parent = parent || this;
  // if (isolated) {
  //   child = new Scope();
  //   child.$root = parent.$root;
  //   child.$$asyncQueue = parent.$$asyncQueue;
  //   child.$$postDigestQueue = parent.$$postDigestQueue;
  //   child.$$applyAsyncQueue = this.$$applyAsyncQueue;
  // } else {
  //   var ChildScope = function() {};
  //   ChildScope.prototype = this;
  //   child = new ChildScope();
  // }
  // parent.$$children.push(child);
  // child.$$watchers = [];
  child.$$listeners = {};
  // child.$$children = [];
  // child.$parent = parent;
  // return child;
};
````
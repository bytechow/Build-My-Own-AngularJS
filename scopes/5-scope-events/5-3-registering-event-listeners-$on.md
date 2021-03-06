### 注册事件监听器：$on
#### Registering Event Listeners: $on

要想在事件触发时得到通知，你需要先注册这个事件。在 AngularJS 中是通过调用 `$on` 来注册事件的。这个函数会接收两个参数：事件名称，还有在事件触发时执行的回调函数。

> AngularJS 中的订阅者被称为 _listener_（监听器），下面我们会继续使用这个名词。

通过 `$on` 注册的监听器既能接收 emit 发出的事件，也能接收 broadcast 广播的事件。实际上，除了事件名称，没有其他方法可以限制要接收的事件。

`$on` 函数实际干了些什么呢？它要把监听器保存在某处，以便之后事件被触发时能够查找到这个监听器。我们会把监听器存储在属性 `$$listeners`（以两个美元符号作为前缀说明这是个私有属性）上。对象里的 key 是事件名称，而 value 是一个存放了该事件监听器的数组。我们可以先测试一下 `$on` 注册的监听器是否按照次序保存了：

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

要在作用域访问到 `$$listeners` 属性，我们先要在构造函数上设置好：

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

在 `$on` 函数内，我们要先检查事件是否已经存在对应的监听器集合了，如果还没有，我们需要初始化一个。初始化之后，我们就可以把新注册的的监听器放到这个集合中来：

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

由于每个监听器在 Scope 树结构中的位置比较重要，我们确实发现了目前 `$$listeners` 的一个小问题：现在所有树结构上的监听器都放在同一个 `$$listeners` 集合中，但我们需要为每一个作用域建立一个独立的 `$$listeners` 集合。

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

这个单元测试失败的原因是虽然 `scope` 和 `child` 都可以访问到 `$$listeners` 集合的，但被隔离的 `isolatedChild` 并不能访问到根作用域上的这个集合。我们需要对子作用域的构造函数进行改造，让它们维护自己的 `$$listeners` 集合。对于非隔离作用域来说，加了这个属性就会屏蔽父作用域上的同名属性，这跟第二章处理 `$$watchers` 时类似：

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
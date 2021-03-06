### 在事件对象中加入当前和目标作用域
#### Including The Current And Target Scopes in The Event Object

现在，我们的事件对象只包含了一个属性：事件名称。下一步，我们需要往里面加入更多的信息。

如果你熟悉浏览器的 DOM 事件，你就会知道它的事件对象包含了两个非常有用的属性：`target`，指向发生当前事件的那个 DOM 元素，`currentTarget`，指向绑定了当前事件处理器的 DOM 元素。

Angular 作用域事件也有类似的一对属性：`targetScope` 指向事件发生在哪个作用域上，而 `currentScope` 指向绑定了当前 listener 的那个作用域。另外，由于可以选择让作用域事件向上或向下传播，这两种传播方式中的两个事件属性也可能是不同的。

||事件起源于|绑定了事件处理器的|
|DOM 事件|target|currentTarget|
|Scope 事件|targetScope|currentScope|

我们从 `targetScope` 开始，它的核心概念就是无论现在是那个 listener 在处理这个事件，它始终指向同一个作用域：

test/scope_spec.js

```js
it('attaches targetScope on $emit', function() {
  var scopeListener = jasmine.createSpy();
  var parentListener = jasmine.createSpy();

  scope.$on('someEvent', scopeListener);
  parent.$on('someEvent', parentListener);
  
  scope.$emit('someEvent');
  
  expect(scopeListener.calls.mostRecent().args[0].targetScope).toBe(scope);
  expect(parentListener.calls.mostRecent().args[0].targetScope).toBe(scope);
});
```

还有针对 `$broadcast` 的测试：

_test/scope_spec.js_

```js
it('attaches targetScope on $broadcast', function() {
  var scopeListener = jasmine.createSpy();
  var childListener = jasmine.createSpy();

  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  
  scope.$broadcast('someEvent');
  
  expect(scopeListener.calls.mostRecent().args[0].targetScope).toBe(scope);
  expect(childListener.calls.mostRecent().args[0].targetScope).toBe(scope);
});
```

要让这两个单元测试通过，我们只需要在 `$emit` 和 `$broadcast` 中，把 `this` 赋值给事件对象作为 target scope 就可以了：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  var event = { name: eventName, targetScope: this };
  // var listenerArgs = [event].concat(_.tail(arguments));
  // var scope = this;
  // do {
  //   scope.$$fireEventOnScope(eventName, listenerArgs);
  //   scope = scope.$parent;
  // } while (scope);
  // return event;
};

Scope.prototype.$broadcast = function(eventName) {
  var event = { name: eventName, targetScope: this };
  // var listenerArgs = [event].concat(_.tail(arguments));
  // this.$$everyScope(function(scope) {
  //   scope.$$fireEventOnScope(eventName, listenerArgs);
  //   return true;
  // });
  // return event;
};
```

与 `scope` 不同，`currentScope` 会根据当前绑定了 listener 的作用域不同而发生变化，它会指向现在正在使用 listener 处理事件的那个作用域。一个证据是，当事件在作用域树中向上或向下传递时，`currentScope` 指向当前事件传播到的作用域。

这种情况下，我们不能使用 Jasmine spy 来进行测试，因为 spy 只能在调用完成之后才能进行验证。`currentScope` 会在遍历作用域时发生变化，所以我们需要在调用 listener 时记录它在那一瞬间的值。我们可以通过 listener 函数加上本地变量来完成这个检测：

对于 `$emit`：

_test/scope_spec.js_

```js
it('attaches currentScope on $emit', function() {
  var currentScopeOnScope, currentScopeOnParent;
  var scopeListener = function(event) {
    currentScopeOnScope = event.currentScope;
  };
  var parentListener = function(event) {
    currentScopeOnParent = event.currentScope;
  };

  scope.$on('someEvent', scopeListener);
  parent.$on('someEvent', parentListener);

  scope.$emit('someEvent');
  
  expect(currentScopeOnScope).toBe(scope);
  expect(currentScopeOnParent).toBe(parent);
});
```

而对于 `$broadcast`，我们这样处理：

_test/scope_spec.js_

```js
it('attaches currentScope on $broadcast', function() {
  var currentScopeOnScope, currentScopeOnChild;
  var scopeListener = function(event) {
    currentScopeOnScope = event.currentScope;
  };
  var childListener = function(event) {
    currentScopeOnChild = event.currentScope;
  };

  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  
  scope.$broadcast('someEvent');
  
  expect(currentScopeOnScope).toBe(scope);
  expect(currentScopeOnChild).toBe(child);
});
```

幸运的是，实现这个功能比这些测试代码要简单直接得多。我们需要做的就只是把我们正遍历到的作用域赋值到事件对象上就可以了：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  // var event = {name: eventName, targetScope: this};
  // var listenerArgs = [event].concat(_.tail(arguments));
  // var scope = this;
  do {
    event.currentScope = scope;
    // scope.$$fireEventOnScope(eventName, listenerArgs);
    // scope = scope.$parent;
  } while (scope);
  return event;
};
Scope.prototype.$broadcast = function(eventName) {
  // var event = {name: eventName, targetScope: this};
  // var listenerArgs = [event].concat(_.tail(arguments));
  this.$$everyScope(function(scope) {
    event.currentScope = scope;
    // scope.$$fireEventOnScope(eventName, listenerArgs);
    // return true;
  });
  return event;
};
```

由于 `currentScope` 代表着当前事件传播的状态，它应该要在事件传播结束后被清空掉。否则，在事件完成传播后，保留该事件对象的程序将会拿到过时的传播状态信息。我们可以通过在一个 listener 中捕获事件对象的方式进行测试，看看当事件传播结束后，`currentScope` 的值是否为 `null`：

_test/scope_spec.js_

```js
it('sets currentScope to null after propagation on $emit', function() {
  var event;
  var scopeListener = function(evt) {
    event = evt;
  };
  scope.$on('someEvent', scopeListener);

  scope.$emit('someEvent');
  
  expect(event.currentScope).toBe(null);
});

it('sets currentScope to null after propagation on $broadcast', function() {
  var event;
  var scopeListener = function(evt) {
    event = evt;
  };
  scope.$on('someEvent', scopeListener);

  scope.$broadcast('someEvent');
  
  expect(event.currentScope).toBe(null);
});
```

我们可以通过在 `$emit` 方法结束遍历时将 `currentScope` 赋值为 `null` 来解决这个问题：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  // var event = {name: eventName, targetScope: this};
  // var listenerArgs = [event].concat(_.tail(arguments));
  // var scope = this;
  do {
    // event.currentScope = scope;
    // scope.$$fireEventOnScope(eventName, listenerArgs);
    // scope = scope.$parent;
  } while (scope);
  event.currentScope = null;
  // return event;
};
```

在 `$broadcast` 方法中，我们也做类似的处理：

```js
Scope.prototype.$broadcast = function(eventName) {
  // var event = {name: eventName, targetScope: this};
  // var listenerArgs = [event].concat(_.tail(arguments));
  // this.$$everyScope(function(scope) {
  //   event.currentScope = scope;
  //   scope.$$fireEventOnScope(eventName, listenerArgs);
  //   return true;
  // });
  event.currentScope = null;
  // return event;
};
```

现在事件监听器就可以知道事件是从作用域树中的哪个节点发出来的，还可以知道当前这个事件被哪个作用域监听。
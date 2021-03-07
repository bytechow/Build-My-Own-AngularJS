### 在事件对象中加入当前和目标作用域
#### Including The Current And Target Scopes in The Event Object

目前事件对象只有一个属性：事件名称。下面，我们要往里面添加更多的信息。

如果你熟悉浏览器的 DOM 事件，你就会知道它的事件对象包含了两个非常有用的属性：`target` 和 `currentTarget`，`target` 代表触发事件的 DOM 元素，`currentTarget` 代表绑定了当前事件处理器的 DOM 元素。由于 DOM 事件是沿着 DOM 树向上传播的，所以这两个属性的值也可能是不一样的。

Angular 作用域事件也有类似的一对属性：`targetScope` 代表触发事件的作用域，而 `currentScope` 代表绑定了当前事件监听器的作用域。另外，由于作用域事件可能向上或向下传播，这两个属性的值也可能是不一样的。

||事件起源于|绑定了事件处理器的|
|-|-|-|
|DOM 事件|target|currentTarget|
|Scope 事件|targetScope|currentScope|

首先是 `targetScope`，无论当前是哪个监听器在处理事件，它都指向同一个作用域。下面是针对 `$emit` 的测试：

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

`$broadcast` 的：

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

要让这两个单元测试通过，我们只需要在 `$emit` 和 `$broadcast` 中，把 `this` 作为 ` targetScope` 赋值给事件对象就可以了：

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

与 `targetScope` 不同，`currentScope` 会随着当前监听器绑定的作用域变化而发生变化，它会指向当前监听器绑定的那个作用域。如果从作用域树的角度来看，当事件向上或向下传播时，`currentScope` 指向的就是当前事件传播到的位置。

在这种情况下，我们就不能再用 Jasmine spy 来测试了，因为 spy 只能验证执行完成后的结果。`currentScope` 会在遍历作用域时不断发生改变，我们需要在调用 listener 时记录它的_瞬时值_。我们可以用 listener 函数结合本地变量来进行测试：

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

幸运的是，实现代码比测试代码要简单得多。我们只需要在遍历时，把作用域赋值到事件对象上就可以了：

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

`currentScope` 用于传达事件传播的当前状态，因此它也应该要在事件传播结束后被清空掉。否则，若有代码缓存了事件对象，在事件传播结束后，它拿到的事件传播状态信息就是过时的。我们可以通过在一个 listener 中捕获事件对象的方式进行测试，看看当事件传播结束后，`currentScope` 的值是否为 `null`：

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

> 译者注：这两个测试用例其实也可以交由 `_.forEach` 生成，减少重复代码。

对于 `$emit`，我们可以在结束遍历后将 `currentScope` 赋值为 `null`：

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

`$broadcast` 也一样：

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

现在事件监听器就可以知道事件是从作用域树的哪个节点发出来的，也可以知道事件正在被哪个作用域监听着。
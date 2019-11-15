### 额外的 listener 参数
#### Additional Listener Arguments

当你 emit 或者 broadcast 一个事件时，一个事件名称往往不足以将目前发生的事情表达出来。将额外的参数与事件关联起来是很常见的做法。我们可以通过在事件名称后面加入任意数量的参数来实现这点。

```js
aScope.$emit('eventName', 'and', 'additional', 'arguments');
```

我需要把这些参数传递到 listener 函数中。首先，`$emit` 和 `$broadcast` 都应该能够接收事件对象中的额外参数：

_test/scope_spec.js_

```js
it('passes additional arguments to listeners on ' + method, function() {
  var listener = jasmine.createSpy();
  scope.$on('someEvent', listener);

  scope[method]('someEvent', 'and', ['additional', 'arguments'], '...');
  
  expect(listener.calls.mostRecent().args[1]).toEqual('and');
  expect(listener.calls.mostRecent().args[2]).toEqual(['additional', 'arguments']);
  expect(listener.calls.mostRecent().args[3]).toEqual('...');
});
```

在 `$emit` 和 `$broadcast` 中，我们会获取接收到的任何额外参数，然后把它们传递给 `$$fireEventOnScope`。我们可以通过调用 Lo-Dash 的 `_.tail` 函数，并传入 `arguments` 对象，来获取额外的参数，这个函数能为我们提供除第一个参数以外的所有参数数组：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  var additionalArgs = _.tail(arguments);
  this.$$fireEventOnScope(eventName, additionalArgs);
};

Scope.prototype.$broadcast = function(eventName) {
  var additionalArgs = _.tail(arguments);
  this.$$fireEventOnScope(eventName, additionalArgs);
};
```

在 `$$fireEventOnScope` 中，我们不能直接就把这个额外参数构成的数组传递给 listener 函数。这是因为 listener 函数希望额外参数就以正常的函数参数形式传递过来就好，而不是一个独立的数组。因此，我们需要利用 [apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) 函数来实现传递事件对象的同时传递额外参数：

_src/scope.js_

```js
Scope.prototype.$$fireEventOnScope = function(eventName, additionalArgs) {
  // var event = { name: eventName };
  var listenerArgs = [event].concat(additionalArgs);
  // var listeners = this.$$listeners[eventName] || [];
  // _.forEach(listeners, function(listener) {
    listener.apply(null, listenerArgs);
  // });
};
```

这样就能满足我们的要求了。
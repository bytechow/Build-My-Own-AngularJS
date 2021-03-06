### 额外的监听器参数
#### Additional Listener Arguments

当你 emit 或者 broadcast 一个事件时，仅凭事件名称不一定足够传达当前所发生的一切。因此，传递与事件关联的额外参数是很常见的。传递事件时除了传入事件名称参数，我们还可以在后面传入任意数量的参数，这些参数就是额外参数。

```js
aScope.$emit('eventName', 'and', 'additional', 'arguments');
```

`$emit` 和 `$broadcast` 都要支持接收这些额外参数，并把它们传递给 listener 函数。在调用 listener 时，要把它们放到事件对象之后传入：

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

在 `$emit` 和 `$broadcast` 中，我们会接收所有额外参数，并把它们传递给 `$$fireEventOnScope`。要获取额外参数，我们可以结合使用用 LoDash 的 `_.tail` 函数和 `arguments` 类数组对象。 `_.tail` 获取数组中除第一个元素以外的全部元素：

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

在 `$$fireEventOnScope` 中，我们可不能直接把这个额外参数数组透传给 listener 函数。这是因为 listener 函数并不希望希望额外参数是以数组形式传递过来的。从 listener 的角度看，它希望额外参数是像普通参数那样逐个传递过来的。因此，我们需要利用 [apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) 方法来同时传递事件对象和额外参数：

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

这样才能满足我们的要求。
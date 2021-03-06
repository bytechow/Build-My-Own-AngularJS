### 返回事件对象
#### Returning The Event Object

`$emit` 和 `$broadcast` 还有一个额外特性——它们会返回自身构建的事件对象，这样事件的发起者就可以在事件结束传递后检查事件对象的状态：

_test/scope\_spec.js_

```js
it('returns the event object on ' + method, function() {
  var returnedEvent = scope[method]('someEvent');
  expect(returnedEvent).toBeDefined();
  expect(returnedEvent.name).toEqual('someEvent');
});
```

实现起来也非常简单——我们直接返回事件对象就可以了：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  // var additionalArgs = _.tail(arguments);
  return this.$$fireEventOnScope(eventName, additionalArgs);
};

Scope.prototype.$broadcast = function(eventName) {
  // var additionalArgs = _.tail(arguments);
  return this.$$fireEventOnScope(eventName, additionalArgs);
};

Scope.prototype.$$fireEventOnScope = function(eventName, additionalArgs) {
  // var event = { name: eventName };
  // var listenerArgs = [event].concat(additionalArgs);
  // var listeners = this.$$listeners[eventName] || [];
  // _.forEach(listeners, function(listener) {
  //   listener.apply(null, listenerArgs);
  // });
  return event;
};
```
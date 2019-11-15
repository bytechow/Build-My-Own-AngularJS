### 事件对象

#### Event Objects

现在我们调用 listener 时没有带上任何参数，但这并不是 Angular 的做法。我们在调用 listener 时需要带上一个_事件对象_（event object）。

作用域中使用到的事件对象就是一个普通的 JavaScript 对象，它包含了与事件相关的信息和行为。我们会在事件对象中加入几个属性，但首先我们需要加入每一个事件都会有的事件名称 `name`。下面关于事件对象的一个测试用例（更确切地说，是两个测试用例）：

_test/scope\_spec.js_

```js
it('passes an event object with a name to listeners on '+method, function() {
  var listener = jasmine.createSpy();
  scope.$on('someEvent', listener);

  scope[method]('someEvent');

  expect(listener).toHaveBeenCalled();
  expect(listener.calls.mostRecent().args[0].name).toEqual('someEvent');
});
```

> Jasmine spy 中的 `call.mostRecent()` 函数包含了最近一次 spy 调用时的信息。它有一个 `args` 属性，存储了当时传递给函数的参数数组。

关于事件对象的一个重要特点就是，我们会将相同的事件对象传递给每一个 listener 函数。应用开发者可以通过在上面添加额外的属性来实现 listener 之间的通信。这个重要性足以保证它有自己的单元测试：

_test/scope\_spec.js_

```js
it('passes the same event object to each listener on '+method, function() {
  var listener1 = jasmine.createSpy();
  var listener2 = jasmine.createSpy();
  scope.$on('someEvent', listener1);
  scope.$on('someEvent', listener2);

  scope[method]('someEvent');

  var event1 = listener1.calls.mostRecent().args[0];
  var event2 = listener2.calls.mostRecent().args[0];

  expect(event1).toBe(event2);
});
```

我们可以在 `$$fireEventOnScope` 函数中构建这个事件对象，然后把它传递给 listener 函数：

_src/scope.js_

```js
Scope.prototype.$$fireEventOnScope = function(eventName) {
  var event = { name: eventName };
  // var listeners = this.$$listeners[eventName] || [];
  // _.forEach(listeners, function(listener) {
    listener(event);
  // });
};
```




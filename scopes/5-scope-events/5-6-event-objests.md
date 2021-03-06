### 事件对象

#### Event Objects

现在我们调用 listener 时没有带上任何参数，但这并不是 Angular 真正的工作方式。实际上，我们会在调用 listener 时带上一个_事件对象_（event object）。

作用域中用到的事件对象就是一个普通的 JavaScript 对象，它携带了与事件相关的信息和行为。我们将会在事件对象中加入几个属性，但首先要加入的属性是 `name` 属性，毕竟每一个事件都必定会有自己的名称。下面是与事件对象的一个测试用例（准确来说，是两个测试用例）：

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

> Jasmine spy 中的 `call.mostRecent()` 记录了最近一次 spy 函数调用时的信息。它的返回值中有一个 `args` 属性，存储了当时传递给函数的参数数组。

事件对象有一个重要特点，就是会将同一个事件对象传递给每一个 listener 函数。应用开发者可以通过在上面添加额外的属性来实现 listener 之间的通信。这个特点非常重要，足以让它拥有专属的单元测试：

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

我们在 `$$fireEventOnScope` 函数中创建事件对象，然后把它传递给 listener 函数就可以了：

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




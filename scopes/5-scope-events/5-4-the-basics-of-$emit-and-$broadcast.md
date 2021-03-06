### $emit 和 $broadcast 的基础
#### The basics of $emit and $broadcast

现在既然注册了监听器，就可以通过触发事件来调用它们了。如上所述，`$emit` 和 `$broadcast` 这两个函数就是为了实现这个功能的。

两个函数的基本功能就是：调用时传入事件名称作为参数，就会触发对应事件的所有监听器。当然，相应地，它们不会调用其他事件的监听器：

_test/scope_spec.js_

```js
it('calls the listeners of the matching event on $emit', function() {
  var listener1 = jasmine.createSpy();
  var listener2 = jasmine.createSpy();
  scope.$on('someEvent', listener1);
  scope.$on('someOtherEvent', listener2);

  scope.$emit('someEvent');

  expect(listener1).toHaveBeenCalled();
  expect(listener2).not.toHaveBeenCalled();
});

it('calls the listeners of the matching event on $broadcast', function() {
  var listener1 = jasmine.createSpy();
  var listener2 = jasmine.createSpy();
  scope.$on('someEvent', listener1);
  scope.$on('someOtherEvent', listener2);

  scope.$broadcast('someEvent');
  
  expect(listener1).toHaveBeenCalled();
  expect(listener2).not.toHaveBeenCalled();
});
```

> 这里我们使用 Jasmine 的 spy 函数作为 listener 函数。他们是一个被称为存根函数（stub functions）的特殊函数，除了记录函数是否被调用过、调用时传入了什么参数，它什么都不会做。利用 spy 函数，我们就能方便地监测到作用域对 listener 做了什么。
> 
> 如果你以前用过 mock 对象或其他类型的测试工具，应该不会对 spy 函数感到陌生。它们也被称为 mock 函数。

目前 `$emit` 和 `$broadcast` 两个函数的行为是相同的，但足以让这些测试通过。这两个函数都能根据事件名称找到对应的 listener 集合，并依次调用这些 listener：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  var listeners = this.$$listeners[eventName] || [];
  _.forEach(listeners, function(listener) {
    listener();
  });
};

Scope.prototype.$broadcast = function(eventName) {
  var listeners = this.$$listeners[eventName] || [];
  _.forEach(listeners, function(listener) {
    listener();
  });
};
```

### $emit 和 $broadcast 的基础功能
#### The basics of $emit and $broadcast

现在，我们已经注册了 listener，就可以触发事件并使用它们。就像我们上面提到的，`$emit` 和 `$broadcast` 就是用于完成这个功能的两个方法。

这两个函数的基本功能就是，当你在调用它们时把事件名称当作参数传递进去，它们就会触发注册了这个事件的所有 listener。当然，相对地，他们不会调用其他事件的 listener：

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

> 我们使用 Jasmine 的 spy 函数来作为 listener 函数。他们是特殊的存根函数（stub functions），除了记录函数是否被调用过、调用时传入了什么参数之外它们什么都不会做。使用 spy 函数，我们就能方便地检查作用域对 listener 做了什么。
> 
> 如果你以前用过 mock 对象或其他类型的测试工具，应该不会对 spy 函数感到陌生。它们也可以被称为 mock 函数。

我们可以通过引入 `$emit` 和 `$broadcast` 函数使这些测试通过，目前这两个函数的行为是相同的。它们会查找到事件名称对应的 listener 函数，并依次调用它们：

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

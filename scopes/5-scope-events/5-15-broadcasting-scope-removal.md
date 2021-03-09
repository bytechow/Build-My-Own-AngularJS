### 作用域被移除时发出广播
#### Broadcasting Scope Removal

在某些情况下，知道一个作用域被移除是很有用的。一个典型案例就是在指令元素即将被销毁时，我们需要处理 DOM 监听器以及其他与此相关的引用。解决方法就是我们在指令作用域上监听一个叫 `$destroy` 的事件。（注意，这个事件名称上带有一个美元符号。这表示这个事件来源于 Angular 框架而不是应用代码。）

那 `$destroy` 事件具体来源于哪里呢？当某人调用作用域的 `$destroy` 函移除这个作用域时，就会发起这个 `$destroy` 事件：

_test/scope_spec.js_

```js
it('fires $destroy when destroyed', function() {
  var listener = jasmine.createSpy();
  scope.$on('$destroy', listener);

  scope.$destroy();
  
  expect(listener).toHaveBeenCalled();
});
```

当作用域被销毁时，它的所有子作用域也会被销毁。因此，子作用域上的监听销毁事件的 listener 也会接收到 `$destroy` 事件：

_test/scope_spec.js_

```js
it('fires $destroy on children destroyed', function() {
  var listener = jasmine.createSpy();
  child.$on('$destroy', listener);

  scope.$destroy();
  
  expect(listener).toHaveBeenCalled();
});
````

我们怎么来实现这个效果呢？实际上，我们已经有一个可以在作用域及其子作用域上触发事件的函数了，也就是 `$broadcast` 了。所以，我们可以在 `$destroy` 函数中直接使用 `$broadcast` 来广播事件：

_src/scope.js_

```js
Scope.prototype.$destroy = function() {
  this.$broadcast('$destroy');
  // if (this.$parent) {
  //   var siblings = this.$parent.$$children;
  //   var indexOfThis = siblings.indexOf(this);
  //   if (indexOfThis >= 0) {
  //     siblings.splice(indexOfThis, 1);
  //   }
  // }
  // this.$$watchers = null;
};
```
### 作用域被移除时发出广播
#### Broadcasting Scope Removal

在某些情况下，我们是需要知道作用域被移除的。比如在指令元素被销毁时，我们需要销毁 DOM 监听器以及其他引用。解决方法就是在指令作用域上监听 `$destroy` 事件。（注意，这个事件名称前有一个美元符号，表明这个事件来源于 Angular 框架而不是应用代码。）

那 `$destroy` 事件是从哪里发起的呢？调用 `$destroy` 方法销毁作用域时，就会触发这个 `$destroy` 事件：

_test/scope_spec.js_

```js
it('fires $destroy when destroyed', function() {
  var listener = jasmine.createSpy();
  scope.$on('$destroy', listener);

  scope.$destroy();
  
  expect(listener).toHaveBeenCalled();
});
```

作用域被销毁时，它所有的子作用域也会被销毁。因此，子作用域也会接收到 `$destroy` 事件：

_test/scope_spec.js_

```js
it('fires $destroy on children destroyed', function() {
  var listener = jasmine.createSpy();
  child.$on('$destroy', listener);

  scope.$destroy();
  
  expect(listener).toHaveBeenCalled();
});
````

该怎么实现这个功能呢？实际上，我们只需要在调用 `$destroy` 方法时调用 `$broadcast` 就好了：

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
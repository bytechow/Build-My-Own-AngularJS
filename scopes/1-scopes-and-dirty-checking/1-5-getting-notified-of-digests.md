### digest 时得到通知（Getting Notified Of Digests）

如果要想在 Angular 启动 digest 时得到通知，你可以利用 watch 函数会在 digest 过程中被调用的这个特点：注册一个只有 watch 函数而没有 listener 函数的 watcher。下面，我们来针对这种情况添加一个测试用例。

_test/scope_spec.js_

```js
it('may have watchers that omit the listener function', function() {
  var watchFn = jasmine.createSpy().and.returnValue('something');
  scope.$watch(watchFn);

  scope.$digest();

  expect(watchFn).toHaveBeenCalled();
});
```

在这个应用场景中，我们不需要在 watch 函数中返回值，虽然我们确实可以返回值，这里展示就是返回了值的情况。当作用域上的 digest 启动后，程序会抛出一个异常。这是因为它正试图调用一个不存在的 listener 函数。要让这个测试用例通过，我们需要在 `$watch` 函数中检查调用时是否少传入了 listener 函数，如果是，那我们就用一个空函数来代替：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn || function() { },
    last: initWatchVal
  };
  this.$$watchers.push(watcher);
};
```

如果你确实需要这样使用 watch 函数，要注意即使没有传入 `listenerFn`，Angular 依然会检查 `watchFn` 的返回值。要是这个 watch 函数返回了某个值，这个值就会受脏值检测机制约束。要确保在这个应用场景中不会产生额外的工作，就不要返回值。这样 watch 函数的返回值就一直是 undefined 了。
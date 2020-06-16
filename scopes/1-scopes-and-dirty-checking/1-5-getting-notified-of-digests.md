### 在 digest 时得到通知（Getting Notified Of Digests）

要想在 Angular 启动 digest 时接收到通知，我们可以利用 watch 函数会在 digest 过程中被调用的这个特点：注册一个只有 watch 函数、没有 listener 函数的 watcher。先来添加一个测试用例：

_test/scope_spec.js_

```js
it('may have watchers that omit the listener function', function() {
  var watchFn = jasmine.createSpy().and.returnValue('something');
  scope.$watch(watchFn);

  scope.$digest();

  expect(watchFn).toHaveBeenCalled();
});
```

实际上，在这个用例中，我们不需要让 watch 函数有返回值（虽然我们确实可以这样做，这里展示就是有返回值的情况）。当作用域上的 digest 启动后，程序会抛出一个异常。这是因为它正试图调用一个不存在的 listener 函数。要让这个测试用例通过，我们需要检查 `$watch` 调用时是否传入了 listener 函数，如果没有传入，那我们就把这个参数变量初始化一个空函数：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  // var watcher = {
  //   watchFn: watchFn,
    listenerFn: listenerFn || function() { },
  //   last: initWatchVal
  // };
  // this.$$watchers.push(watcher);
};
```

如果你确实需要这样来使用 watch 函数，要注意即使没有传入 `listenerFn`，Angular 依然会检查 `watchFn` 的返回值。要是这个 watch 函数返回了某个值，这个值就会成为要进行脏值检测的对象之一。为了确保 watch 函数不会产生额外的运算，我们最好不要返回任何值，这样 watch 函数的返回值就一直都是 `undefined` 了。
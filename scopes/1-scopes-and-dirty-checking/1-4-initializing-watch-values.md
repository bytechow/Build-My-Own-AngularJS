### 初始化侦听数据值（Initializing Watch Values）

大多数情况下，我们使用 watch 函数的返回值与保存在 `last` 变量中的旧值进行比较就可以满足需求了，但在第一次调用 watch 函数时这样处理也适用吗？此时，我们尚未对 `last` 变量进行赋值，因此它的值还是 `undefined`。那如果目标数据正好也是 `undefined` 的时候就有问题了。我们希望在第一次 watch 函数执行时，listener 函数就被调用，但由于目前没有考虑到侦听数据初始值为 `undefined` 的情况，这时程序会认为值没有发生“改变”，也就不会调用 listener 函数了：

_test/scope_spec.js_

```js
it('calls listener when watch value is first undefined', function() {
  scope.counter = 0;
  
  scope.$watch(
    function(scope) { return scope.someValue; },
    function(newValue, oldValue, scope) { scope.counter++; }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

在这个测试用例中，listener 函数需要被调用一次才符合我们的需求。我们需要做的就是把 `last` 属性初始化为一个保证是唯一的值，这样无论 watch 函数返回什么值，包括 `undefined`，这个值都不会与初始的 `last` 属性值相等。

使用一个函数作为 `last` 属性的初始值就能满足我们的需求了，因为 JavaScript 函数是所谓的_引用值_（reference values）——它们不与其他任何值相等，只与自身相等。我们在 `scope.js` 的顶层作用于中加入一个函数值：

_src/scope.js_

```js
function initWatchVal() { }
```

现在，我们就可以在生成新的 watcher 时把这个函数赋值给它的 `last` 属性了：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn,
    last: initWatchVal
  };
  this.$$watchers.push(watcher);
};
```

这样处理后，新创建的 watcher 就始终可以调用 listener 函数了，无论它的 watch 函数返回什么值。

同时，`initWatchVal` 也会作为 watcher 的旧值传递给 listener 处理函数。但我们并不希望这个特殊函数能在 `scope.js` 以外的地方访问到。对于新创建的 watcher，我们应该把新值当作是旧值传入：

_test/scope_spec.js_

```js
it('calls listener with new value as old value the first time', function() {
  scope.someValue = 123;
  var oldValueGiven;

  scope.$watch(
    function(scope) { return scope.someValue; },
    function(newValue, oldValue, scope) { oldValueGiven = oldValue; }
  );

  scope.$digest();
  expect(oldValueGiven).toBe(123);
});
```

在 `$digest` 方法中，当我们调用 listener 时，我们会判断当前采用的旧值是不是约定的初始值，如果是则替换为新值：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var self = this;
  // var newValue, oldValue;
  // _.forEach(this.$$watchers, function(watcher) {
  //   newValue = watcher.watchFn(self);
  //   oldValue = watcher.last;
  //   if (newValue !== oldValue) {
  //     watcher.last = newValue;
      watcher.listenerFn(newValue,
        (oldValue === initWatchVal ? newValue : oldValue),
        self);
  //   } 
  // });
};
```
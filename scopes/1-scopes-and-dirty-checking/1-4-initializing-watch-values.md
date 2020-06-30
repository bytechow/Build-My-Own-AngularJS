### 侦听数据的初始值（Initializing Watch Values）

在大多数情况下，我们使用 watch 函数的返回值与 `last` 属性中保存的旧值进行比较就可以满足脏值检测的需求了，但在第一次调用 watch 函数时这样处理合适吗？第一次调用时，`last` 属性还未被赋值，它的值还是 `undefined`。这时，如果要侦听的变量值恰好也是 `undefined` 就会产生问题了。我们希望在第一次 watch 函数执行时，listener 函数无论如何都会被调用，由于我们目前尚未考虑到侦听数据初始值为 `undefined` 的情况，当前的代码会判断值没有发生改变，也就不会调用 listener 函数了：

_test/scope\_spec.js_

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

在这个测试用例中，我们希望 listener 函数会至少会被调用一次。该怎么办呢？我们可以把 `last` 属性初始化为一个保证是唯一的值，这样无论 watch 函数返回什么值（包括 `undefined`），这个值都不会与初始的 `last` 属性值相等。

其实把一个函数作为 `last` 属性的初始值就能满足我们的需求了，因为 JavaScript 函数是所谓的_引用值_（reference values）——它们与其他任何值均不相等，只与自身相等。在 `scope.js` 的顶层上下文中声明一个函数：

_src/scope.js_

```js
function initWatchVal() { }
```

在生成新的 watcher 时，我们会把这个函数赋值给它的 `last` 属性了：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  // var watcher = {
  //   watchFn: watchFn,
  //   listenerFn: listenerFn,
    last: initWatchVal
  // };
  // this.$$watchers.push(watcher);
};
```

这样在创建 watcher 后（执行 $digest 时），无论 watch 函数返回什么值，listener 函数都会被调用。

与此同时，`initWatchVal` 也会作为 watcher 最初的旧值被传递到 listener 函数中。但我们并不希望这个特殊函数能在 `scope.js` 以外的地方被访问到。要实现这个目的，我们只需要在 watcher 注册后，把新值作为 listener 旧值的实参传入就可以了：

_test/scope\_spec.js_

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

在 `$digest` 方法中，在调用 listener 时，我们会判断当前的旧值是否是约定的初始值，如果是就把它替换为新值传入：

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




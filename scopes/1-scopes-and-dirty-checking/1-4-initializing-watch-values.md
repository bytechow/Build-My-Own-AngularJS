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
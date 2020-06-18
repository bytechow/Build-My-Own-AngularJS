### 处理值为 NaN 的情况（NaNs）

在 JavaScript 里面，`NaN`（Not-a-Number）是不等于自身的。这听上去很奇怪，但事实就是这样。如果我们不在脏值检测函数中处理 `NaN`，那么侦听值为 `NaN` 的 watcher 将一直被认为是“脏”的。

> NaN 的相等性特性实际上来自 IEEE 对浮点数字约定的标准，这套标准也应用在 JavaScript 以外的很多编程语言上。可以看看 Stack Overflow 上这个[有趣的讨论](https://stackoverflow.com/questions/1565164/what-is-the-rationale-for-all-comparisons-returning-false-for-ieee754-nan-values)。

对于基于值的脏值检测，我们使用的 LoDash `isEqual` 函数已经帮我们处理了这个问题了。但对于基于引用的脏值检测来说，就需要我们自己来解决这个问题。我们可以使用下面这个单元测试来验证：

_test/scope_spec.js_

```js
it('correctly handles NaNs', function() {
  scope.number = 0/0; // NaN
  scope.counter = 0;
  
  scope.$watch(
    function(scope) { return scope.number; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

我们会对一个运算结果值是 `NaN` 的数据进行侦听，当它发生变化时计数器就自增 1。我们希望这个计数器在第一次 `$digest` 时增加一次，之后就保持不变。但实际我们会发现这个测试抛出了”TTL reached“的异常。这是因为 `NaN` 不自等，新值与旧值一直都被判定为不相等，使得作用域的状态一直无法稳定下来。

我们可以通过修改 `$$areEqual` 函数来修复这个问题：

_src/scope.js_

```js
Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
  // if (valueEq) {
  //   return _.isEqual(newValue, oldValue);
  // } else {
    return newValue === oldValue ||
    (typeof newValue === 'number' && typeof oldValue === 'number' &&
     isNaN(newValue) && isNaN(oldValue));
  // } 
};
```

现在，值为 `NaN` 的 watcher 的表现也符合我们的预期了。
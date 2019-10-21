### 侦听非集合数据的变化
#### Detecting Non-Collection Changes

`$watchCollection` 的目标就是要侦听数组和对象。但是，它也能支持 watch 函数返回值是一个非集合的情况。在这种情况下，它会回退到直接调用 `$watch` 的工作状态。虽然这可能是 `$watchCollection` 中最乏味的内容，但先开发这个能充实我们的函数结构。

下面这个测试用于确认函数的基本行为：

_test/scope_spec.js_

```js
it('works like a normal watch for non-collections', function() {
  var valueProvided;

  scope.aValue = 42;
  scope.counter = 0;
  
  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      valueProvided = newValue;
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  expect(valueProvided).toBe(scope.aValue);

  scope.aValue = 43;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```
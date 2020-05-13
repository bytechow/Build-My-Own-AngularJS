### 输入表达式跟踪
#### Input Tracking

关于表达式侦听，我们可以做一个优化，也就是_输入表达式跟踪_（input tracking）。它的核心概念就是当一个表达式是由一个或多个_输入表达式_组成（比如 `'a * b'` 就是由 `'a'` 和 `'b'`组成），除非其中一个输入表达式发生了变化，否则无需再重新计算表达式。

举个例子，如果有一个数组字面量，它里面所有的元素都没有发生变化：

_test/scope_spec.js_

```js
it('does not re-evaluate an array if its contents do not change', function() {
  var values = [];

  scope.a = 1;
  scope.b = 2;
  scope.c = 3;
  
  scope.$watch('[a, b, c]', function(value) {
    values.push(value);
  });
  
  scope.$digest();
  expect(values.length).toBe(1);
  expect(values[0]).toEqual([1, 2, 3]);
  
  scope.$digest();
  expect(values.length).toBe(1);
  scope.c = 4;
  
  scope.$digest();
  expect(values.length).toBe(2);
  expect(values[1]).toEqual([1, 2, 4]);
});
```
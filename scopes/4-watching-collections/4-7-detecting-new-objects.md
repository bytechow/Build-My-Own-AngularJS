### 侦听转换为对象的情况
#### Detecting New Objects

下面我们把注意力放到对象上，或者更准确地说，是放到除了数组和类数组对象以外的对象上。这一般指的是类似这样的字典值：

```js
{
  aKey: 'aValue',
  anotherKey: 42
}
```

我们检测对象中发生变化的方式跟数组类似。但对象会更简单一点，因为并没有 "类对象对象" 这类数据来干扰我们。而另一方面，我们需要在侦测变化中做更多的工作，因为对象并没有数组的 `length` 便捷属性。

像数组一样，我们会先确保能处理以往不是对象的值被赋值为一个对象的情况：

_test/scope_spec.js_

```js
it('notices when the value becomes an object', function() {
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);

  scope.obj = { a: 1 };
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们可以沿用之前处理数组的方案来解决这个问题。如果旧值不是一个真正的对象，我们就把它转换为一个对象，并记录一次变化：

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  // newValue = watchFn(scope);

  if (_.isObject(newValue)) {
    if (isArrayLike(newValue)) {
      // if (!_.isArray(oldValue)) {
      //   changeCount++;
      //   oldValue = [];
      // }
      // if (newValue.length !== oldValue.length) {
      //   changeCount++;
      //   oldValue.length = newValue.length;
      // }
      // _.forEach(newValue, function(newItem, i) {
      //   var bothNaN = _.isNaN(newItem) && _.isNaN(oldValue[i]);
      //   if (!bothNaN && newItem !== oldValue[i]) {
      //     changeCount++;
      //     oldValue[i] = newItem;
      //   }
      // });
    } else {
      if (!_.isObject(oldValue) || isArrayLike(oldValue)) {
        changeCount++;
        oldValue = {};
      }
    }
  } else {
    // if (!self.$$areEqual(newValue, oldValue, false)) {
    //   changeCount++;
    // }
    // oldValue = newValue;
  }
  
  return changeCount;
};
```

注意，由于数组也是一种对象，我们不能单单使用 `_.Object` 来对旧值进行检测，还需要使用 `_.isArrayLike` 来排除数组和类数组对象。
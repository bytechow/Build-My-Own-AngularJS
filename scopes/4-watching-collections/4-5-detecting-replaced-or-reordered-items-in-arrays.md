### 侦测数组元素的替换和重排序
#### Detecting Replaced or Reordered Items in Arrays

我们还需要处理数组的另一种变化，也就是在不改变数组长度的前提下进行数组元素的替换或者重新排序：

_test/scope_spec.js_

```js
it('notices an item replaced in an array', function() {
  scope.arr = [1, 2, 3];
  scope.counter = 0;
 
  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
 
  scope.$digest();
  expect(scope.counter).toBe(1);
 
  scope.arr[1] = 42;
  scope.$digest();
  expect(scope.counter).toBe(2);
 
  scope.$digest();
  expect(scope.counter).toBe(2);
});

it('notices items reordered in an array', function() {
  scope.arr = [2, 1, 3];
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.arr.sort();
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
````

要检测出这种变化，我们需要遍历整个数组，对每一个索引位置上的元素进行比较。这样做就能把无论是通过替换还是重排序发生变化的元素都找出来了。在遍历的同时，我们也要将内部的 `oldValue` 数组元素内容跟新数组进行同步：

```js
var internalWatchFn = function(scope) {
  newValue = watchFn(scope);
  
  if (_.isObject(newValue)) {
    if (_.isArray(newValue)) {
      // if (!_.isArray(oldValue)) {
      //   changeCount++;
      //   oldValue = [];
      // }
      // if (newValue.length !== oldValue.length) {
      //   changeCount++;
      //   oldValue.length = newValue.length;
      // }
      _.forEach(newValue, function(newItem, i) {
        if (newItem !== oldValue[i]) {
          changeCount++;
          oldValue[i] = newItem;
        }
      });
    } else {

    }
  } else {
    // if (!self.$$areEqual(newValue, oldValue, false)) {
    //   changeCount++;
    // }
    // oldValue = newValue;
  }

  // return changeCount;
};
```

这里我们使用 Lo-Dash 的 `_.forEach` 方法来对新数组进行遍历。这个方法在每次迭代都能为我们提供数组元素和它所在的位置索引。我们用这个位置索引来访问旧数组对应位置的值。

第一章我们已经看到由于 `NaN` 不自等引发的问题。我们已经在普通的 watcher 中对这个值进行特殊处理，下面这个单元测试就展示了 Angular 也会在针对集合数据的 watcher 中进行相同的处理：

_test/scope_spec.js_

```js
it('does not fail on NaNs in arrays', function() {
  scope.arr = [2, NaN, 3];
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

这个测试会抛出一个异常，这是因为 NaN 值的存在会一直触发变化，从而产生一个无限的 digest
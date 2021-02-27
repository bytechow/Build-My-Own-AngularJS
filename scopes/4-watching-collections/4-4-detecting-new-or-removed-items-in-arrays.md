### 侦测数组元素的新增或移除
#### Detecting New Or Removed Items in Arrays

跟数组相关的第二类变化是数组元素的新增或移除，这种操作会导致数组长度发生变化。下面我们分别为这两种操作添加一个单元测试：

_test/scope_spec.js_

```js
it('notices an item added to an array', function() {
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
  
  scope.arr.push(4);
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});

it('notices an item removed from an array', function() {
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
  
  scope.arr.shift();
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

在这两个单元测试中，我们都会对作用域上的数组属性进行侦听，并且会对数组内容进行操作。我们会验证在数组操作后的下一次 digest 能否检测到数组发生的变化。

我们可以通过比较新旧值的数组长度来检测是否发生了这种变化。同时我们必须将 `oldValue` 的长度与当前数组长度进行同步，只要将当前数组的长度赋值给旧数组的长度属性就可以：

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  // newValue = watchFn(scope);

  if (_.isObject(newValue)) {
    if (_.isArray(newValue)) {
      // if (!_.isArray(oldValue)) {
      //   changeCount++;
      //   oldValue = [];
      // }
      if (newValue.length !== oldValue.length) {
        changeCount++;
        oldValue.length = newValue.length;
      }
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
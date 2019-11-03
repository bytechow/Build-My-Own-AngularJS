### 侦听对象属性的移除
#### Detecting Removed Attributes in Objects

我们还需需要侦测的对象变化就是属性的移除：

_test/scope_spec.js_

```js
it('notices when an attribute is removed from an object', function() {
  scope.counter = 0;
  scope.obj = { a: 1 };

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);

  delete scope.obj.a;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

对于数组，我们可以通过对内部数组进行截断的方式来实现元素的移除（具体是通过对数组的 `length` 属性进行赋值）然后同时对两个数组进行遍历，看看对应的所有元素是不是都一样。但这种方法没法用在对象上。要验证对象上的属性是否被移除掉，我们需要使用另一个循环。在这个循环中，我们会对旧对象的属性进行遍历，然后看看在新数组中是否还存在这个属性。如果不存在，说明它被移除了，我们需要把它从旧对象中也移除掉。

> 译者注：这是因为属性可能在初始化时被显式赋值为 `undefined`，而如果移除这个属性之后，再访问这个属性，结果值一样是 `undefined`，所以如果只是使用相等性判断的话，我们就无法区分属性是否存在还是被移除。

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  newValue = watchFn(scope);
  
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
      // if (!_.isObject(oldValue) || isArrayLike(oldValue)) {
      //   changeCount++;
      //   oldValue = {};
      // }
      // _.forOwn(newValue, function(newVal, key) {
      //   var bothNaN = _.isNaN(newVal) && _.isNaN(oldValue[key]);
      //   if (!bothNaN && oldValue[key] !== newVal) {
      //     changeCount++;
      //     oldValue[key] = newVal;
      //   }
      // });
      _.forOwn(oldValue, function(oldVal, key) {
        if (!newValue.hasOwnProperty(key)) {
          changeCount++;
          delete oldValue[key];
        }
      });
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
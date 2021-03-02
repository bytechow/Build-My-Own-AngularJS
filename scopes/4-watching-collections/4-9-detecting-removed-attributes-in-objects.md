### 侦听对象属性的移除
#### Detecting Removed Attributes in Objects

关于对象变化，最后要讨论的就是移除对象属性的情况了：

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

对于数组，我们可以通过截断（就是对数组的 `length` 属性进行重新赋值）的方式来处理元素移除的情况，对两个数组进行同步遍历，看看对应位置的元素的值是否还一样。但如果要验证对象上的属性是否被移除掉，就不能用这种方法，我们需要采用另一种循环的方式。在循环中，我们会遍历旧对象的属性，看看各个属性是否还存在于在新数组中。如果不存在，说明这个属性已经被移除了，我们需要把它从旧对象中移除掉。

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
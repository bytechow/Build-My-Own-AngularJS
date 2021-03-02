### 侦听对象属性的新增或替换
#### Detecting New Or Replaced Attributes in 

我们希望 `$watchCollection` 也能把对象新增属性的情况视作对象发生了变更：

_test/scope_spec.js_

```js
it('notices when an attribute is added to an object', function() {
  scope.counter = 0;
  scope.obj = {a: 1};

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.obj.b = 2;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

属性值发生变化也一样：

_test/scope_spec.js_

```js
it('notices when an attribute is changed in an object', function() {
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
  
  scope.obj.a = 2;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们可以跟处理数组一样，遍历新值里的属性，然后看旧值相同位置上的值是否相同：

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
      _.forOwn(newValue, function(newVal, key) {
        if (oldValue[key] !== newVal) {
          changeCount++;
          oldValue[key] = newVal;
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

当我们在遍历的时候，我们也会把新对象的属性值赋值给旧对象对应属性，以便之后再次用它进行比较。

> LoDash 的 `_.forOwn` 函数会对对象成员属性进行便利，但只会遍历定义在这个对象上的属性，排除其他原型链上的属性。也就是说，`$watchCollection` 不会对原型链上继承下来的属性进行监听。

同样，我们也需要对 `NaN` 值进行特殊处理。若对象上有一个属性的值为 `NaN`，就会让程序进入无限的 digest 循环：

_test/scope_spec.js_

```js
it('does not fail on NaN attributes in objects', function() {
  scope.counter = 0;
  scope.obj = {a: NaN};

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

若新旧两个值都是 `NaN` 值，我们就认为它们是相同的：

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
      _.forOwn(newValue, function(newVal, key) {
        var bothNaN = _.isNaN(newVal) && _.isNaN(oldValue[key]);
        if (!bothNaN && oldValue[key] !== newVal) {
          // changeCount++;
          // oldValue[key] = newVal;
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
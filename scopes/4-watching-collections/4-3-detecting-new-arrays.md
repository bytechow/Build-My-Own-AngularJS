### 侦测转换为数组的情况
#### Detecting New Arrays

内嵌的 watch 函数将会有两个顶层的条件分支：一个是用于处理对象的，两一个是处理对象以外的类型的。由于在 JavaScript 中数组也属于一种对象，它们会在第一个分支中进行处理，但在这个纷至中，我们需要在内嵌一个条件分支，用于区分处理数组和非数组的对象。

目前我们可以直接使用 Lo-Dash 的 `_.Object` 和 `_.isArray` 函数来区分值到底算是对象还是数组：

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  // newValue = watchFn(scope);

  if (_.isObject(newValue)) {
    if (_.isArray(newValue)) {

    } else {

    }
  } else {
    // if (!self.$$areEqual(newValue, oldValue, false)) {
    //   changeCount++;
    // }
    // oldValue = newValue;
  }
  
  // return changeCount;
}
````

我们确认这个对象就是数组的方式，就是在确认是对象之后马上再加一个判断看这个值是否是数组类型的。如果不是，那显然已经发生了改变：

_test/scope_spec.js_

```js
it('notices when the value becomes an array', function() {
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.arr = [1, 2, 3];
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

由于现在 `internalWatchFn` 处理数组的分支还什么都没有，我们没法察觉到这种变化。要处理这种情况，我们需要先对旧值的类型进行判断：

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  // newValue = watchFn(scope);

  if (_.isObject(newValue)) {
    if (_.isArray(newValue)) {
      if (!_.isArray(oldValue)) {
        changeCount++;
        oldValue = [];
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

如果旧值不是一个数组，我们就记录为一次变化。同时，我们也会把旧值初始化为一个空数组。稍后，我们才会在这个内部数组中对正在侦听的数组内容进行拷贝，但现在这样已经可以通过我们的测试了。
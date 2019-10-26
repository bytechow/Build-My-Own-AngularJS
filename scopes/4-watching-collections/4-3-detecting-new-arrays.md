### 侦测新数组
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
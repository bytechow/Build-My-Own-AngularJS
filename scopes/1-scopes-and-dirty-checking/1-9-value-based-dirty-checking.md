### 基于值的脏值检测（Value-Based Dirty-Checking）

目前我们使用全等运算符 `===` 来对新旧值进行比较，由于这种方法能够既对所有原始数据类型（数字、字符串等）进行比较，又能检查出对象或数组是否发生引用指向变化，所以它能满足大多数的应用场景。但 Angular 还有一种检查变化的方式，这种方式能把发生在对象或数组内部的变化也能检测出来。也就是，我们不仅可以对引用指向上的变化进行侦听，还能对（引用数据的）值本身的变化进行侦听。

这种脏值检测可以通过在调用 `$watch` 函数时传入一个布尔值标识作为第三个参数来进行配置。当这个标识是 `true` 的时候，就会启动给予值的脏值检测。下面我们加入一个验证这个功能的单元测试：

_test/scope_spec.js_

```js
it('compares based on value if enabled', function() {
  scope.aValue = [1, 2, 3];
  scope.counter = 0;

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    },
    true
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.aValue.push(4);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

在这个测试中，`scope.aValue` 数组发生变化时计数器会加 1。在我们对数组添加一个元素后，我们希望脏值检测能检测到这一变化。但事实上没有达成这个效果，因为 `scope.aValue` 依然是同一个数组，只是内容发生了变化。

我们首先要重新定义 `$watch` 函数，让它能接收布尔值标识，并把标识保存在 watcher 中：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  var watcher = {
    // watchFn: watchFn,
    // listenerFn: listenerFn || function() { },
    valueEq: !!valueEq,
  //   last: initWatchVal
  };
  // this.$$watchers.push(watcher);
  // this.$$lastDirtyWatch = null;
};
```

这里我们会通过使用两次取反的方法来把这个标识强制转换为真正的布尔值。如果使用者在调用 `$watch` 时没有带上第三个参数，`valueEq` 就是 `undefined`，它赋值到 watcher 对象的属性时就变成 `false` 了。

基于值的脏值检测实际上就是，如果旧值或新值是对象或数组，我们就要对它们包含的元素进行遍历。如果发现两个值内的某个内容元素不相等，那说明 watcher 变“脏”了。如果在要比较的值中还嵌套了其他对象或数组，我们也会通过递归对这些嵌套对象进行基于值的比较。

Angular 本身实现了一个[相等性比较函数](https://github.com/angular/angular.js/blob/8d4e3fdd31eabadd87db38aa0590253e14791956/src/Angular.js#L812)，但我们会使用 [LoDash 的相等性比较函数](https://lodash.com/docs/4.17.15#isEqual)来代替，因为对于本书要实现的内容来说，它就足够了。然后，我们会定一个新方法，这个方法会根据 `valueEq` 这个布尔值标识来决定使用哪种比较方式：

_src/scope.js_

```js
Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
  if (valueEq) {
    return _.isEqual(newValue, oldValue);
  } else {
    return newValue === oldValue;
  }
};
```

要发现值内发生的变化，我们还需要改变在每个 watcher 中保存旧值的方法。只是保存当前值的引用是不够的，因为这时对值内部任何的改变都同步到我们目前持有的引用。这样的话，由于 `$$areEqual` 永远都只会拿到对同一个值的两个引用，因此无法发现它们内部发生了什么变化。因此，我们需要对当前值做一份深拷贝，然后把这份拷贝当作旧值来保存。

除了相等性比较的方法，Angular 也实现了自己的[深拷贝方法](https://github.com/angular/angular.js/blob/8d4e3fdd31eabadd87db38aa0590253e14791956/src/Angular.js#L725)，而我们会使用 [LoDash 的深拷贝方法](https://lodash.com/docs/4.17.15#cloneDeep) 代替。

在 `$$digestOnce` 函数中，我们该用用 `$$areEqual` 方法进行比较，同时在有需要时（需要进行基于值的比较时）进行深拷贝，并赋值给 `last` 变量：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  // var newValue, oldValue, dirty;
  // _.forEach(this.$$watchers, function(watcher) {
  //   newValue = watcher.watchFn(self);
  //   oldValue = watcher.last;
    if (!self.$$areEqual(newValue, oldValue, watcher.valueEq)) {
      // self.$$lastDirtyWatch = watcher;
      watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
      // watcher.listenerFn(newValue,
      //   (oldValue === initWatchVal ? newValue : oldValue),
      //   self);
      // dirty = true;
  //   } else if (self.$$lastDirtyWatch === watcher) {
  //     return false;
  //   }
  // });
  // return dirty;
};
```

现在我们就能支持两种相等性比较的方式了，而单元测试自然也通过了。

基于值的比较显然比基于引用的比较更为复杂，有时候两者简直是天差地别。递归遍历一个嵌套的数据结构要花不少时间，而保存它的一个深拷贝版本也会占用不少内存。这也是为什么 Angular 不会默认进行基于值的脏值检测，我们需要显式地传入标识参数才能启用这种脏值检测方式。

> 其实 Angular 还提供第三种脏值检测方式：集合侦听。我们会在第三章实现这种方式。

我们还需要处理一个 JavaScript 的“怪癖”才能算完成基于值的比较功能。
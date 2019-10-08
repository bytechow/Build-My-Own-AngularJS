### 多个变化共用同一个 listener 函数
#### Watching Several Changes With One Listener: $watchGroup

到目前为止，我们看到所有的 watch 函数和 listener 函数是一个简单的因果关系对：当这个发生了变化，就那样做。但还有一种情况也并不少见，就是同时观察多个状态，当其中一个状态发生改变时就执行某段代码。

鉴于 Angular watch 就是一个普通的 JavaScript 函数，我们目前的 watch 代码就能完美支持这种情况了：我们可以在 watch 函数对多个要侦听的数据进行查询，最后返回一个包含所有查询结果的复合值就可以了，这个复合值发生的变化就可以触发 listener 函数了。

> 译者注：当然这里说的 watcher 需要传入第三个参数值 `true`，开启基于值的比较模式才可以检测复合值内容是否发生变化。

事实上，在 Angular 1.3 以后的版本中我们已经不再需要手动创建这类函数，可以直接使用 Angular 内建的 `$watchGroup`。

_test/scope_spec.js_


```js
describe('$watchGroup', function() {
  
  var scope;
  beforeEach(function() {
    scope = new Scope();
  });

  it('takes watches as an array and calls listener with arrays', function() {
    var gotNewValues, gotOldValues;
    
    scope.aValue = 1;
    scope.anotherValue = 2;
    
    scope.$watchGroup([
      function(scope) { return scope.aValue; },
      function(scope) { return scope.anotherValue; }
    ], function(newValues, oldValues, scope) {
      gotNewValues = newValues;
      gotOldValues = oldValues;
    });
    scope.$digest();
    
    expect(gotNewValues).toEqual([1, 2]);
    expect(gotOldValues).toEqual([1, 2]);
  });
});
```

在这个测试中，我们把 listener 函数中传入的 `newValue` 和 `oldValue` 抓取出来，然后检查它的值是否就是包含两个 watch 函数返回值的数。

下面我们先来尝试第一种开发 `$watchGroup` 的方式，就是在注册每一个 watcher 的时候都重用同一个 listener 函数：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  var self = this;
  _.forEach(watchFns, function(watchFn) {
    self.$watch(watchFn, listenerFn);
  });
};
```

但这并没有解决问题。我们希望 listener 接收到的是包含全部 watcher 结果值的数组，但现在每次 listener 被调用时接收到的只是单个 watch 函数的返回值。

我们需要为每一个 watch 都分别定义一个内部 listener 函数，然后在内部的 listener 函数中就把这个 watch 的结果值更新数组对应位置中。然后我们就可以把这个数组传递给原始的 listener 函数了。我们会使用两个数组，一个用于存放新值，一个用于存放旧值：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  // var self = this;
  var newValues = new Array(watchFns.length);
  var oldValues = new Array(watchFns.length);
  _.forEach(watchFns, function(watchFn, i) {
    self.$watch(watchFn, function(newValue, oldValue) {
      newValues[i] = newValue;
      oldValues[i] = oldValue;
      listenerFn(newValues, oldValues, self);
    });
  });
};
```

> `$watchGroup` 用的是中是基于引用的变化侦测方式。

我们开发的第一个版本的 `$watchGroup` 的问题在于太急切地想要调用 listener 函数了：如果在 watch 数组（watchFns）的几个数据都发生了变化，那么 listener 也会被调用多次，但我们希望在这样的情况下它只会被调用一次。更糟的是，因为一旦发现其中一个数据发生变化就会调用一次 listener，那这时候就很可能在 `oldValues` 和 `newValues` 数组中产生新旧值的混淆，最终会让用户看到结果值出现前后不一致的问题。

我们下面来测试在发生多个数据变化的情况下，listener 函数是否会只被调用一次：

_test/scope_spec.js_

```js
it('only calls listener once per digest', function() {
  var counter = 0;

  scope.aValue = 1;
  scope.anotherValue = 2;
  
  scope.$watchGroup([
    function(scope) { return scope.aValue; },
    function(scope) { return scope.anotherValue; }
  ], function(newValues, oldValues, scope) {
    counter++;
  });
  scope.$digest();
  
  expect(counter).toEqual(1);
});
```

那我们怎么把 listener 延迟到所有 watcher 都被检查只之后调用呢？由于 `$watchGroup` 并不负责运行 digest，我们无法在 `$watchGroup` 中找到调用 listener 的好位置。但我们可以利用上几节中实现的 `$evalAsync`。它的作用就是让任务延迟执行，但执行时间还是会在同一个 digest 周期之内——这正是我们需要的。

我们在 `$watchGroup` 中创建一个新的内部函数，叫做 `$watchGroupListener`。这个函数就负责调用原始的 listener 函数并传入之前的两个数组。然后在每一个内部 listener 函数中，如果之前没有设定执行调用 listener 的定时任务，我们就设定一个：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  var self = this;
  var oldValues = new Array(watchFns.length);
  var newValues = new Array(watchFns.length);
  var changeReactionScheduled = false;

  function watchGroupListener() {
    listenerFn(newValues, oldValues, self);
    changeReactionScheduled = false;
  }
  
  _.forEach(watchFns, function(watchFn, i) {
    self.$watch(watchFn, function(newValue, oldValue) {
      newValues[i] = newValue;
      oldValues[i] = oldValue;
      if (!changeReactionScheduled) {
        changeReactionScheduled = true;
        self.$evalAsync(watchGroupListener);
      }
    });
  });
};
```
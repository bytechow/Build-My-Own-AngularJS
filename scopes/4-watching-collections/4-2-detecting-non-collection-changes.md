### 侦听非集合数据的变化
#### Detecting Non-Collection Changes

`$watchCollection` 的目标就是要侦听数组和对象。但是，它也能支持 watch 函数返回值是一个非集合的情况。在这种情况下，它会回退到直接调用 `$watch` 的工作状态。虽然这可能是 `$watchCollection` 中最乏味的内容，但在处理这个情况的同时能充实我们的函数结构。

下面这个测试用于确认函数的基本行为：

_test/scope_spec.js_

```js
it('works like a normal watch for non-collections', function() {
  var valueProvided;

  scope.aValue = 42;
  scope.counter = 0;
  
  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      valueProvided = newValue;
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  expect(valueProvided).toBe(scope.aValue);

  scope.aValue = 43;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们使用 `$watchCollection` 对作用域上一个数字类型的属性进行侦听。在 listener 函数中，我们会让计数器加一，同时捕获新值，把它保存到一个局部变量中。然后我们断言，这个 watcher 会调用 listener 函数，就像普通的、不针对集合数据的 watcher 那样。

> 我们暂时先忽略 `oldValue` 参数。在 `$watchCollection` 语境下，它需要进行一些特殊的处理，我们会在本章后面的部分再对它进行讨论。

在调用 watch 函数时，`$watchCollection` 首先会调用 _原始的_ watch 函数来获取我们想要侦听的值。然后，它会检查该值与之前的值相比是否有变化，并把该值作为下一次 digest 周期的旧值：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  var newValue;
  var oldValue;

  var internalWatchFn = function(scope) {
    newValue = watchFn(scope);
  
    // Check for changes
  
    oldValue = newValue;
  };
  
  var internalListenerFn = function() {
  };
  
  return this.$watch(internalWatchFn, internalListenerFn);
};
```

通过把 `newValue` 和 `oldValue` 的变量定义放到 `internalListenerFn` 函数体的外面，我们就能同时在 `internalWatchFn` 函数和 `internalListenerFn` 函数共用这两个变量。它们也会通过 `$watchCollection` 函数形成的闭包在 digest 的不同轮次之间一直保持着。这对于旧值来说就特别重要，因为我们需要在不同轮次中比较这个值。

这个 listener 函数要做的只是直接调用传入的 listener 函数，把新旧值和作用域都传进去就可以了：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  var self = this;
  // var newValue;
  // var oldValue;

  // var internalWatchFn = function(scope) {
  //   newValue = watchFn(scope);
  
  //   // Check for changes
  
  //   oldValue = newValue;
  // };
  
  // var internalListenerFn = function() {
    listenerFn(newValue, oldValue, self);
  // };
  
  // return this.$watch(internalWatchFn, internalListenerFn);
};
```

回想一下，`$digest` 对是否需要调用 listener 函数的判断，是通过对 watch 函数在连续两轮 digest 中返回的值进行比较得出的。但是现在内部的 watch 函数还什么都没有返回，listener 函数自然也不会被调用了。

那内部的 watch 函数应该返回什么呢？既然在 `$watchCollection` 外部已经无法访问这个函数，我们也无需做太大的改变。我们只需要知道一旦发生了改变，那连续两轮返回的值就会不同，而这就是 listener 函数的触发条件。

Angular 解决这个问题的方法是引入一个整数计数器，然后只要检测到有一个值发生了变化就对它进行递增。每一个通过 `$watchCollection` 注册的 watcher 都有自己的计数器，它会在 watcher 的整个生命周期中不断地递增。只要在 `internalWatchFn` 中返回这个计数器，我们就确保能满足对 watch 函数的约束
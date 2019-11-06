### 把旧的集合值传递给 listener 函数
#### Handing The Old Collection Value To Listeners

我们 约定 watcher 的 listener 函数可以接收到三个参数：watch 函数的最新值，watch 函数上一次的返回值，还有作用域对象。我们在这章的开发中也遵从了传递这三个参数的约定，但我们传递的值是有问题的，尤其是在 watch 函数上一次的返回值上。


问题出在我们现在是在 `innerWatchFn` 上保存旧值的，当我们调用 listener 函数时，这个值已经被更新为新值了。因此传递给 listener 函数的值一直都是一样的。这就是非集合数据遇到的情况，所以下面这个单元测试就无法通过了：

_test/scope_spec.js_

```js
it('gives the old non-collection value to listeners', function() {
  scope.aValue = 42;
  var oldValueGiven;

  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      oldValueGiven = oldValue;
    }
  );
  
  scope.$digest();
  
  scope.aValue = 43;
  scope.$digest();
  
  expect(oldValueGiven).toBe(42);
});
```

数组也会遇到一样的问题：

_test/scope_spec.js_

```js
it('gives the old array value to listeners', function() {
  scope.aValue = [1, 2, 3];
  var oldValueGiven;

  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      oldValueGiven = oldValue;
    }
  );

  scope.$digest();

  scope.aValue.push(4);
  scope.$digest();

  expect(oldValueGiven).toEqual([1, 2, 3]);
});
```

还有对象：

_test/scope_spec.js_

```js
it('gives the old object value to listeners', function() {
  scope.aValue = { a: 1, b: 2 };
  var oldValueGiven;

  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      oldValueGiven = oldValue;
    }
  );

  scope.$digest();
  
  scope.aValue.c = 3;
  scope.$digest();
  
  expect(oldValueGiven).toEqual({ a: 1, b: 2 });
});
```

基于值的比较与拷贝的代码运行正常，也能有效地协助完成变化侦测的任务，因此我们并不想对这部分代码进行更改。我们会引入一个新的变量，这个变量会在各个 digest 迭代的过程一直存在。我们把这个变量命名为 `veryOldValue`，它保存了一份旧集合值的拷贝，这份拷贝不会在 `internalWatchFn` 中发生变化。

要生成 `veryOldValue` 就需要拷贝数组或者对象，代价是比较昂贵的。但我们之前已经做了很多就是为了避免每次在集合表中都复制完整的集合。因此我们只会在真的要使用 `veryOldValue` 的时候才生成。我们可以通过 listener 函数传入的参数是否至少有两个来对此进行判断：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  // var self = this;
  // var newValue;
  // var oldValue;
  // var oldLength;
  var veryOldValue;
  var trackVeryOldValue = (listenerFn.length > 1);
  // var changeCount = 0;

  // ...
  
};
````
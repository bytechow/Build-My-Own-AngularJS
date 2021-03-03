### 把旧集合值传递给 listener 函数
#### Handing The Old Collection Value To Listeners

我们约定 watcher 的 listener 函数可以接收到三个参数：watch 函数返回的新值，watch 函数上一次返回的值（旧值）和作用域对象。我们开发 `$watchCollection` 的过程中也遵守了这个约定，但现在传递的值是有问题的，尤其是旧值。

原因是因为我们现在是在 `innerWatchFn` 上维护旧值的，在调用 listener 函数之前，这个值已经被更新为新值了。因此传递给 listener 函数的值一直都是一样的。下面这个单元测试展示 `$watchCollection` 侦听非集合数据时会遇到的问题：

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

之前比较与拷贝的代码运行正常，也能有效满足变化侦测的需求，因此我们并不打算动这部分代码。相反，我们会引入一个名为 `veryOldValue` 的局部变量，这个变量会在不同的 digest 轮次的过程一直存在，它保存了一份旧集合值的拷贝，这份拷贝不会在 `internalWatchFn` 中发生变化。
  
要维护 `veryOldValue` 就得拷贝数组或者对象，这个操作的成本比较高。但我们之前已经做了很多就是为了避免每次在集合表中都复制完整的集合。因此我们只会在真的要使用 `veryOldValue` 的时候才生成。我们可以通过 listener 函数传入的参数是否至少有两个来对此进行判断：

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

`Function` 的 `length` 属性包含了函数中声明的参数数量。如果多于一个，例如`(newValue, oldValue)` 或者 `(newValue, oldValue, scope)`，我们才会启用并跟踪这个 `veryOldValue`。

请注意，除非你在 listener 函数参数中声明了 `oldValue`，否则 `$watchCollection` 不用承担拷贝 `veryOldValue` 的成本。这也意味着你不能想当然地从 listener 函数的 `arguments` 对象中访问 `oldValue`。要访问的话就必须声明它。

剩下来的工作就需要在 `internalListenerFn` 中完成了。我们不再传递 `oldValue` 给 listener，而会传递 `veryOldValue`。然后需要基于当前值拷贝下一个 `veryOldValue` 值以便下一次使用。我们可以使用 `_.clone` 对集合数据进行浅拷贝，它同样适用于原始数据类型（primitives）：

_src/scope.js_

```js
var internalListenerFn = function() {
  listenerFn(newValue, veryOldValue, self);

  if (trackVeryOldValue) {
    veryOldValue = _.clone(newValue);
  }
};
```

在第一章中，我们说在第一次调用 listener 函数时 `oldValue` 应该与新值是相同的。这个规则对于 `$watchCollection` 中的 listner 函数来说也是一样的：

_test/scope_spec.js_

```js
it('uses the new value as the old value on first digest', function() {
  scope.aValue = { a: 1, b: 2 };
  var oldValueGiven;

  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      oldValueGiven = oldValue;
    }
  );

  scope.$digest();
  
  expect(oldValueGiven).toEqual({ a: 1, b: 2 });
});
```

传入的 oldValue 是 `undefined`，导致这个单元测试未能通过，这是由于在 listener 函数第一次调用之前，我们还未对 `veryOldValue` 进行过赋值。

我们需要设置一个标识来区分当前是否在 listener 的第一次调用中，然后根据这个标识来选用不同的调用方式：

src/scope.js

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  // var self = this;
  // var newValue;
  // var oldValue;
  // var oldLength;
  // var veryOldValue;
  // var trackVeryOldValue = (listenerFn.length > 1);
  // var changeCount = 0;
  var firstRun = true;

  // ...
  
  var internalListenerFn = function() {
    if (firstRun) {
      listenerFn(newValue, newValue, self);
      firstRun = false;
    } else {
      // listenerFn(newValue, veryOldValue, self);
    }

    // if (trackVeryOldValue) {
    //   veryOldValue = _.clone(newValue);
    // }
  };
  
  // return this.$watch(internalWatchFn, internalListenerFn);
};
```
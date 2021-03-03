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

之前的用于比较和拷贝的代码运行正常，也能满足侦测变化的需求，因此我们并不打算动这部分代码。相反，我们会引入一个名为 `veryOldValue` 的局部变量来解决这个问题，这个变量能被不同的 digest 轮次访问到，它保存了一份旧集合值的拷贝，这份拷贝不会在 `internalWatchFn` 中发生变化。
  
要维护 `veryOldValue` 就得拷贝数组或者对象，这个操作的成本比较高。但我们做了这么多，不就是为了在侦听时不用每次都拷贝完整的集合数据吗？因此我们只会在真的要用到 `veryOldValue` 的时候，才会维护它。我们可以通过 listener 函数是否传入了两个以上参数来判断要不要使用这个变量（进行跟踪）：

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

`Function` 的 `length` 属性能告诉我们函数调用时实际传入的参数数量。如果多于一个，例如 `(newValue, oldValue)` 或者 `(newValue, oldValue, scope)`，我们才会使用这个 `veryOldValue` 进行跟踪。

这意味着除非我们在调用 listener 函数时声明了 `oldValue` 参数，否则 `$watchCollection` 不需要承担拷贝 `veryOldValue` 的成本。这也意味着我们不能想当然地觉得可以从 listener 函数的 `arguments` 对象中访问到 `oldValue`，要访问的话就必须先声明它。

剩下来的工作就需要在 `internalListenerFn` 中完成了。我们传递给 listener 的旧值参数不再是 `oldValue`，而是 `veryOldValue`。接下来，我们会把当前的新值赋值给 `veryOldValue` 以便下一次使用。我们可以利用 `_.clone` 方法来获取集合数据的浅拷贝，这个方法同样适用于对原始数据类型（primitives）进行拷贝：

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
### 类数组对象
#### Array-Like Objects

上面我们已经针对数组进行了处理，但还需要考虑一种特殊情况。

除了普通数组，即从 `Array` 原型继承下来的数组以外，JavaScript 一些“长”得像数组但并不是真的是数组的对象。Angular 的 `$watchCollection` 也可以像处理数组一样地处理这些对象，我们希望自己的框架也能做到。

每个函数都会有的 [arguments](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions_and_function_scope/arguments) 局部变量就是这样一个类数组对象，它包含了调用函数时传入的所有参数。下面，我们来写一个测试看看 `$watchCollection` 是否能处理这个类数组对象：

```js
it('notices an item replaced in an arguments object', function() {
  (function() {
    scope.arrayLike = arguments;
  })(1, 2, 3);
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.arrayLike; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  
  scope.arrayLike[1] = 42;
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

在测试用例中，我们创建了一个自执行的匿名函数，然后将 `arguments` 变量赋值到作用域上的一个属性上，这样作用域上就有一个 `arguments` 类数组对象了。然后可以看看 `$watchCollection` 能否监测这个对象上发生的变化。

DOM [NodeList](https://developer.mozilla.org/en-US/docs/Web/API/NodeList) 也是一种类数组对象，是通过特定 DOM 接口获取到的，例如 `querySelectorAll` 和 `getElementsByTagName`。我们也为这类象建一个用例测试。

_test/scope_spec.js_

```js
it('notices an item replaced in a NodeList object', function() {
  document.documentElement.appendChild(document.createElement('div'));
  scope.arrayLike = document.getElementsByTagName('div');

  scope.counter = 0;
  
  scope.$watchCollection(
    function(scope) { return scope.arrayLike; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
  
  document.documentElement.appendChild(document.createElement('div'));
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们先往 DOM 结构上添加一个 `<div>`，然后通过调用 `document` 上的 `getElementsByTagName` 来获取关于 `<div>` 的 `NodeList` 对象。我们把这个对象放到作用域上，并对它进行侦听。要想触发 `NodeList` 的变化，我们只需要往 DOM 里面多添加一个 `<div>` 就好了，因为这个列表是所谓的 "实时集合"，新增后马上就会更新这个对象。我们要验证 `$watchCollection` 能否检测到这个变化。

结果是两个单元测试都未能通过。问题出在 Lo-Dash 的 `_.isArray` 函数上，它只能适用于真正的数组类型而不包括其他类数组对象。我们需要为这种用例创建一个新的判断函数：

_src/scope.js_

```js
function isArrayLike(obj) {
  if (_.isNull(obj) || _.isUndefined(obj)) {
    return false;
  }
  var length = obj.length;
  return _.isNumber(length);
}
```

这个函数会接收一个对象，然后返回一个布尔值，这个布尔值指明了这个对象到底是不是类数组对象。具体来说，我们会通过判断这个对象存在且有一个数字类型的 `length` 属性值来确定它到底是不是类数组对象。这个函数还未完善，但已经满足现在的需求的。

现在我们需要做的是在 `$watchCollection` 中使用这个新的判断函数来代替 `_.isArray`：

_src/scope.js_

```js
var internalWatchFn = function(scope) {
  // newValue = watchFn(scope);

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

注意，当我们处理任何一种类数组对象时，内部的 `oldValue` 数组一直是一个真正的数组，而不是类数组对象。

> 其实 String（字符串）也满足类数组对象的要求，因为它也有一个 `length` 属性，并且为每一个字符都提供了索引属性。然而，JavaScript `String` 并不是一个 JavaScript `Object`，因此在外层的 `_.isObject` 判断条件已经防止了字符串被当作集合了。也就是说，`$watchCollection` 把字符串视作非集合。

由于 JavaScript 字符串的不可变性，我们不能改变它的内容，因此把字符串当作集合来进行侦听也没有多大用处。
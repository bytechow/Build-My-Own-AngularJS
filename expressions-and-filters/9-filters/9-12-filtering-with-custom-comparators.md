### 使用自定义比较器进行过滤
#### Filtering With Custom Comparators

如果你想自定义过滤时如何对两个值进行比较的逻辑，我们也可以往过滤器中传入一个自定义的_比较器_（comparator）函数作为第二个参数。举个例子，下面我们自定义了一个比较器，用于对两个值进行全等比较 `===`：

```js
it('allows using a custom comparator', function() {
  var fn = parse('arr | filter:{$: "o"}:myComparator');
  expect(fn({
    arr: ['o', 'oo', 'ao', 'aa'],
    myComparator: function(left, right) {
      return left === right;
    }
  })).toEqual(['o']);
});
```

这跟本节中之前提到过的提供过滤器判定函数并不一样。过滤器判定函数会根据传入的任意条件决定是否需要对输入值进行滤，而比较器函数会把传入值和过滤值（或部分过滤值）进行比较，然后决定他们是否应该进行比对。

我们需要在 filter 函数接收这第三个参数，然后把它传递给 `createPredicateFn` 用于创建判定函数：

_src/filter_filter.js_

```js
function filterFilter() {
  return function(array, filterExpr, comparator) {
    // var predicateFn;
    // if (_.isFunction(filterExpr)) {
    //   predicateFn = filterExpr;
    // } else if (_.isString(filterExpr) ||
    //   _.isNumber(filterExpr) || _.isBoolean(filterExpr) || _.isNull(filterExpr) || _.isObject(filterExpr)) {
      predicateFn = createPredicateFn(filterExpr, comparator);
  //   } else {
  //     return array;
  //   }
  //   return _.filter(array, predicateFn);
  // };
}
```

现在在 `createPredicateFn` 函数中，只有在没有传入比较器函数时才会生成一个新的比较器函数：

_src/filter_filter.js_

```js
function createPredicateFn(expression, comparator) {
  // var shouldMatchPrimitives = 
  // _.isObject(expression) && ('$' in expression);
  
  if (!_.isFunction(comparator)) {
    comparator = function(actual, expected) {
      // if (_.isUndefined(actual)) {
      //   return false;
      // }
      // if (_.isNull(actual) || _.isNull(expected)) {
      //   return actual === expected;
      // }
      // actual = ('' + actual).toLowerCase();
      // expected = ('' + expected).toLowerCase();
      // return actual.indexOf(expected) !== -1;
    };
  }
  // return function predicateFn(item) {
  //   if (shouldMatchPrimitives && !_.isObject(item)) {
  //     return deepCompare(item, expression.$, comparator);
  //   }
  //   return deepCompare(item, expression, comparator, true);
  // };
}
```
### 使用其他原始类型的值进行过滤
#### Filtering With Other Primitives

传递给过滤器的表达式不一定是一个字符串。它也可能是一个数字：

_test/filter_filter_spec.js_

```js
it('filters with a number', function() {
  var fn = parse('arr | filter:42');
  expect(fn({
    arr: [
      { name: 'Mary', age: 42 }, { name: 'John', age: 43 }, { name: 'Jane', age: 44 }
    ]
  })).toEqual([
    { name: 'Mary', age: 42 }
  ]);
});
```

或者是一个布尔值：

_test/filter_filter_spec.js_

```js
it('filters with a boolean value', function() {
  var fn = parse('arr | filter:true');
  expect(fn({
    arr: [
      { name: 'Mary', admin: true }, { name: 'John', admin: true }, { name: 'Jane', admin: false }
    ]
  })).toEqual([
    { name: 'Mary', admin: true }, { name: 'John', admin: true }
  ]);
});
```

我们也需要为这个类型的表达式创建对应的判定函数：

_src/filter_filter.js_

```js
function filterFilter() {
  // return function(array, filterExpr) {
  //   var predicateFn;
  //   if (_.isFunction(filterExpr)) {
  //     predicateFn = filterExpr;
    } else if (_.isString(filterExpr) ||
               _.isNumber(filterExpr) ||
               _.isBoolean(filterExpr)) {
  //     predicateFn = createPredicateFn(filterExpr);
  //   } else {
  //     return array;
  //   }
  //   return _.filter(array, predicateFn);
  // };
}
```

我们在比较函数中把这些值都强制转换为字符串就好了：

_src/filter_filter.js_

```js
function comparator(actual, expected) {
  actual = ('' + actual).toLowerCase();
  expected = ('' + expected).toLowerCase();
  // return actual.indexOf(expected) !== -1;
}
```

值得注意的是，使用数字（或布尔值）进行过滤并不意味着需要对元素进行相等性判断。这里会把传入的任何值变成一个字符串，因此包含了这个数字的字符串也会被匹配到，效果像下面这个（已经通过了的）测试用例一样：

_test/filter_filter_spec.js_

```js
it('filters with a substring numeric value', function() {
  var fn = parse('arr | filter:42');
  expect(fn({ arr: ['contains 42'] })).toEqual(['contains 42']);
});
```

我们也可以过滤 `null` 值。这时，只有元素值确实是 `null` 的才会被返回，包含 `'null'` 这个子字符串的字符串并不会被匹配到：

_test/filter_filter_spec.js_

```js
it('filters matching null', function() {
  var fn = parse('arr | filter:null');
  expect(fn({ arr: [null, 'not null'] })).toEqual([null]);
});
```

反过来，如果我们要过滤的是字符串 `'null'`，不会匹配会值为 `null` 的元素。只会匹配到（符合条件的）字符串：

_test/filter_filter_spec.js_

```js
it('does not match null value with the string null', function() {
  var fn = parse('arr | filter:"null"');
  expect(fn({ arr: [null, 'not null'] })).toEqual(['not null']);
});
```

我们需要为传入值为 `null` 的表达式创建一个判定函数：

_src/filter_filter.js_

```js
function filterFilter() {
  // return function(array, filterExpr) {
  //   var predicateFn;
  //   if (_.isFunction(filterExpr)) {
  //     predicateFn = filterExpr;
  //   } else if (_.isString(filterExpr) ||
  //              _.isNumber(filterExpr) ||
  //              _.isBoolean(filterExpr) ||
               _.isNull(filterExpr)) {
  //     predicateFn = createPredicateFn(filterExpr);
  //   } else {
  //     return array;
  //   }
  //   return _.filter(array, predicateFn);
  // };
}
```

在比较函数（comparator）中，我们会对 `null` 进行特殊处理。如果实际值、期望值两者之一是 `null`，那么只有当另一方为 `null` 时才会被认为是匹配的：

_src/filter_filter.js_

```js
function comparator(actual, expected) {
  if (_.isNull(actual) || _.isNull(expected)) {
    return actual === expected;
  }
  // actual = ('' + actual).toLowerCase();
  // expected = ('' + expected).toLowerCase();
  // return actual.indexOf(expected) !== -1;
}
```

当在数组中有 `undefined` 的元素值，它们也不应该匹配到字符串 `undefined`：

_test/filter_filter_spec.js_

```js
it('does not match undefined values', function() {
  var fn = parse('arr | filter:"undefined"');
  expect(fn({ arr: [undefined, 'undefined'] })).toEqual(['undefined']);
});
```
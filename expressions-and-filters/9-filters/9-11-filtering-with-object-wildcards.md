### 使用对象通配符进行过滤
#### Filtering With Object Wildcards

如果你想要“让这个对象中的任意属性都匹配这个值”，我们就要在标准对象中使用一个特殊的通配符号 `$` 了：

_test/filter_filter_spec.js_

```js
it('filters with a wildcard property', function() {
  var fn = parse('arr | filter:{$: "o"}');
  expect(fn({
    arr: [
      { name: 'Joe', role: 'admin' }, { name: 'Jane', role: 'moderator' }, { name: 'Mary', role: 'admin' }
    ]
  })).toEqual([
    { name: 'Joe', role: 'admin' }, { name: 'Jane', role: 'moderator' }
  ]);
});
```

与常规的对象属性的不同之处在于，使用通配符属性之后过滤器也会对嵌套对象进行匹配——在任意层级上：

_test/filter_fitler_spec.js_

```js
it('filters nested objects with a wildcard property', function() {
  var fn = parse('arr | filter:{$: "o"}');
  expect(fn({
    arr: [
      { name: { first: 'Joe' }, role: 'admin' }, { name: { first: 'Jane' }, role: 'moderator' }, { name: { first: 'Mary' }, role: 'admin' }
    ]
  })).toEqual([
    { name: { first: 'Joe' }, role: 'admin' }, { name: { first: 'Jane' }, role: 'moderator' }
  ]);
});
```

目前这个属性跟其他原始类型过滤标准并没什么两样。那为什么使用 `{$: "o"}` 而不直接用 "o" 呢？主要原因是当我们在内嵌的对象标准中使用通配符时，这种方法能把通配符的上下文指定为它的父级：

_test/filter_filter_spec.js_

```js
it('filters wildcard properties scoped to parent', function() {
  var fn = parse('arr | filter:{name: {$: "o"}}');
  expect(fn({
    arr: [
      { name: { first: 'Joe', last: 'Fox' }, role: 'admin' },
      { name: { first: 'Jane', last: 'Quick' }, role: 'moderator' },
      { name: { first: 'Mary', last: 'Brown' }, role: 'admin' }
    ]
  })).toEqual([
    { name: { first: 'Joe', last: 'Fox' }, role: 'admin' },
    { name: { first: 'Mary', last: 'Brown' }, role: 'admin' }
  ]);
});
```

当我们对期望对象的内容进行遍历时，我们需要检查这个 key 是否是 `$`。如果是，我们就要对实际对象的_整体_进行匹配，而不单单是对它本身包含的 key 进行匹配（这需要实际对象中包含 `$` 属性）：

_src/filter_filter.js_

```js
return _.every(
  // _.toPlainObject(expected),
  // function(expectedVal, expectedKey) {
  //   if (_.isUndefined(expectedVal)) {
  //     return true;
  //   }
    var isWildcard = (expectedKey === '$');
    var actualVal = isWildcard ? actual : actual[expectedKey];
    return deepCompare(actualVal, expectedVal, comparator);
  // }
);
```

除此之外，我们确实需要在实际对象中匹配任意层次的属性。这就是通配符想要达到的效果。因此，我们需要在调用 `deepCompare` 时传递第四个参数：

_src/filter_filter.js_

```js
return _.every(
  // _.toPlainObject(expected),
  // function(expectedVal, expectedKey) {
  //   if (_.isUndefined(expectedVal)) {
  //     return true;
  //   }
  //   var isWildcard = (expectedKey === '$');
  //   var actualVal = isWildcard ? actual : actual[expectedKey];
    return deepCompare(actualVal, expectedVal, comparator, isWildcard);
  // }
);
```

实际上，当我们在标准对象的顶层使用通配符后，它也能够对数组中的原始类型元素进行匹配：

_test/filter_filter_spec.js_

```js
it('filters primitives with a wildcard property', function() {
  var fn = parse('arr | filter:{$: "o"}');
  expect(fn({ arr: ['Joe', 'Jane', 'Mary'] })).toEqual(['Joe']);
});
```

在判定函数里面，如果是要匹配一个非对象类型的数据，就会直接使用原始过滤器表达式中的 `$` 属性对应的值（如果有的话）进行匹配：

_src/filter_filter.js_

```js
function createPredicateFn(expression) {
  var shouldMatchPrimitives =
    _.isObject(expression) && ('$' in expression);

  // function comparator(actual, expected) {
  //   if (_.isUndefined(actual)) {
  //     return false;
  //   }
  //   if (_.isNull(actual) || _.isNull(expected)) {
  //     return actual === expected;
  //   }
  //   actual = ('' + actual).toLowerCase();
  //   expected = ('' + expected).toLowerCase();
  //   return actual.indexOf(expected) !== -1;
  // }
  
  // return function predicateFn(item) {
    if (shouldMatchPrimitives && !_.isObject(item)) {
      return deepCompare(item, expression.$, comparator);
    }
    // return deepCompare(item, expression, comparator, true);
  // };
}
````

最后，通配符属性也可以嵌套使用。使用以后，代表我们需要实际对象的某个内嵌层中需要存在某个值：

_test/filter_filter_spec.js_

```js
it('filters with a nested wildcard property', function() {
  var fn = parse('arr | filter:{$: {$: "o"}}');
  expect(fn({
    arr: [
      { name: { first: 'Joe' }, role: 'admin' }, { name: { first: 'Jane' }, role: 'moderator' }, { name: { first: 'Mary' }, role: 'admin' }
    ]
  })).toEqual([
    { name: { first: 'Joe' }, role: 'admin' }
  ]);
});
```

目前也会匹配到 `role: 'moderator'`，但我们希望它只会匹配到两个层次都包含 'o' 的对象，因为我们已经在标准对象中指明了。

我们可以通过在给 `deepComare` 传入第四个参数 `inWildcard` 来解决这个问题。我们只会在匹配了一个通配符之后再进行递归时才把它设置为 true：

_src/filter_filter.js_

```js
function deepCompare(
  actual, expected, comparator, matchAnyProperty, inWildcard) {
  // if (_.isString(expected) && _.startsWith(expected, '!')) {
  //   return !deepCompare(actual, expected.substring(1),
  //   }
  //   if (_.isArray(actual)) {
  //     comparator,
  //     matchAnyProperty);
  //   return _.some(actual, function(actualItem) {
  //       return deepCompare(actualItem, expected,
  //       });
  //   }
  //   comparator, matchAnyProperty);
  // if (_.isObject(actual)) {
  //   if (_.isObject(expected)) {
  //     return _.every(_.toPlainObject(expected), function(expectedVal, expectedKey) {
  //       if (_.isUndefined(expectedVal)) {
  //         return true;
  //       }
        var isWildcard = (expectedKey === '$');
        var actualVal = isWildcard ? actual : actual[expectedKey];
        return deepCompare(actualVal, expectedVal,
          comparator, isWildcard, isWildcard);
  //     });
  //   } else if (matchAnyProperty) {
  //     return _.some(actual, function(value, key) {
  //       return deepCompare(value, expected, comparator, matchAnyProperty);
  //     });
  //   } else {
  //     return comparator(actual, expected);
  //   }
  // } else {
  //   return comparator(actual, expected);
  // }
}
```
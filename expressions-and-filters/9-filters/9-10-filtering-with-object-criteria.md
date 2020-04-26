### 使用对象进行过滤
#### Filtering With Object Criteria

当我们需要过滤一个对象数组时，只能使用原始类型对它进行过滤可能还是不太好用。例如，你可能希望过滤某个特定的键值对。

实际上，我们可以提供一个对象作为筛选的标准。下面我们会在输入的对象数组中查找一个 `name` 属性值中包含 `'o'` 字符串的对象。这个对象也包含了 `role` 属性，但过滤器会把他们忽略掉：

_test/filter_filter_spec.js_

```js
it('filters with an object', function() {
  var fn = parse('arr | filter:{name: "o"}');
  expect(fn({
    arr: [
      { name: 'Joe', role: 'admin' },
      { name: 'Jane', role: 'moderator' }
    ]
  })).toEqual([
    { name: 'Joe', role: 'admin' }
  ]);
});
```

当你在过滤对象（作为过滤标准的对象）中指定几个属性值过滤标准时，只有所有属性都符合过滤标准的对象才会被返回：

_test/filter_filter_spec.js_

```js
it('must match all criteria in an object', function() {
  var fn = parse('arr | filter:{name: "o", role: "m"}');
  expect(fn({
    arr: [
      { name: 'Joe', role: 'admin' },
      { name: 'Jane', role: 'moderator' }
    ]
  })).toEqual([
    { name: 'Joe', role: 'admin' }
  ]);
});
```

因此，当过滤对象为空时，所有的对象都能通过这个过滤器：

_test/filter_filter_spec.js_

```js
it('matches everything when filtered with an empty object', function() {
  var fn = parse('arr | filter:{}');
  expect(fn({
    arr: [
      { name: 'Joe', role: 'admin' },
      { name: 'Jane', role: 'moderator' }
    ]
  })).toEqual([
    { name: 'Joe', role: 'admin' },
    { name: 'Jane', role: 'moderator' }
  ]);
});
```

这个标准对象还可能内嵌了对象，我们同样支持对对象进行任意深度的匹配：

_test/filter_filter_spec.js_

```js
it('filters with a nested object', function() {
  var fn = parse('arr | filter:{name: {first: "o"}}');
  expect(fn({
    arr: [
      { name: { first: 'Joe' }, role: 'admin' },
      { name: { first: 'Jane' }, role: 'moderator' }
    ]
  })).toEqual([
    { name: { first: 'Joe' }, role: 'admin' }
  ]);
});
```

同时，就像前面过滤字符串时那样，我们同样可以使用 `!` 前缀来对结果进行取反：

_test/filter_filter_spec.js_

```js
it('allows negation when filtering with an object', function() {
  var fn = parse('arr | filter:{name: {first: "!o"}}');
  expect(fn({
    arr: [
      { name: { first: 'Joe' }, role: 'admin' },
      { name: { first: 'Jane' }, role: 'moderator' }
    ]
  })).toEqual([
    { name: { first: 'Jane' }, role: 'moderator' }
  ]);
});
```

这为我们现在需要实现的东西提供了一个很好的初始测试工具。首先，我们要需要在传入过滤对象时，为它创建一个判定函数：

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
               _.isNull(filterExpr) ||
               _.isObject(filterExpr)) {
  //     predicateFn = createPredicateFn(filterExpr);
  //   } else {
  //     return array;
  //   }
  //   return _.filter(array, predicateFn);
  // };
}
```

在 `deepCompare` 中，如果我们传入一个对象作为期望值，我们不会把它直接与实际值进行比较。我们需要做一些别的：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator) {
  // if (_.isString(expected) && _.startsWith(expected, '!')) {
  //   return !deepCompare(actual, expected.substring(1), comparator);
  // }
  // if (_.isObject(actual)) {
    if (_.isObject(expected)) {
      
    } else {
      // return _.some(actual, function(value, key) {
      //   return deepCompare(value, expected, comparator);
      // });
    }
  // } else {
  //   return comparator(actual, expected);
  // }
}
```

我们需要对这个对象类型的期望值进行遍历，然后与实际对象的相应值进行深度比较。如果发现实际对象完全符合期望值的标准，才认为是匹配的：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator) {
  // if (_.isString(expected) && _.startsWith(expected, '!')) {
  //   return !deepCompare(actual, expected.substring(1), comparator);
  // }
  // if (_.isObject(actual)) {
  //   if (_.isObject(expected)) {
      return _.every(
        _.toPlainObject(expected),
        function(expectedVal, expectedKey) {
          return deepCompare(actual[expectedKey], expectedVal, comparator);
        }
      );
  //   } else {
  //     return _.some(actual, function(value, key) {
  //       return deepCompare(value, expected, comparator);
  //     });
  //   }
  // } else {
  //   return comparator(actual, expected);
  // }
}
```

这种实现方式同样兼容匹配标准对象为嵌套对象的情况，因为递归调用 `deepCompare` 会再次检查内嵌的值是否为一个对象。

> 这里我们对过滤标准对象额外使用了 `toPlainObject`。如果标准对象刚好又继承某些原型属性，这个方法能“打平”继承属性，让每个继承属性同样接收检查。

如果标准对象中有某些值是 `undefined`，这些值会被忽略：

_test/filter_filter_spec.js_

```js
it('ignores undefined values in expectation object', function() {
  var fn = parse('arr | filter:{name: thisIsUndefined}');
  expect(fn({
    arr: [
      { name: 'Joe', role: 'admin' },
      { name: 'Jane', role: 'moderator' }
    ]
  })).toEqual([
    { name: 'Joe', role: 'admin' },
    { name: 'Jane', role: 'moderator' }
  ]);
});
```

要实现这个效果，当发现过滤标准对象中的属性值为 `undefined`，就直接认为这个属性是匹配的就可以了：

_src/filter_filter.js_

```js
return _.every(
  // _.toPlainObject(expected),
  // function(expectedVal, expectedKey) {
    if (_.isUndefined(expectedVal)) {
      return true;
    }
  //   return deepCompare(actual[expectedKey], expectedVal, comparator);
  // }
);
```

如果对象中有嵌套的数组，只要数组中有一个元素是符合过滤规则的，都认为匹配上了。标准对象在这里会“跳”了一个层次，因此我们不需要做任何特殊的处理也能让它匹配到数组里面的元素：

_test/fitler_filter_spec.js_

```js
it('filters with a nested object in array', function() {
  var fn = parse('arr | filter:{users: {name: {first: "o"}}}');
  expect(fn({
    arr: [
      { users: [{ name: { first: 'Joe' }, role: 'admin' }, { name: { first: 'Jane' }, role: 'moderator' }] },
      { users: [{ name: { first: 'Mary' }, role: 'admin' }] }
    ]
  })).toEqual([{
    users: [{ name: { first: 'Joe' }, role: 'admin' },
      { name: { first: 'Jane' }, role: 'moderator' }
    ]
  }]);
});
```

我们需要在 `deepCompare` 中加入一个特殊的例外判断。如果我们发现实际值为数组，我们需要递归处理每个数组元素，只要有其中一个元素匹配到了都返回 `true`：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator) {
  // if (_.isString(expected) && _.startsWith(expected, '!')) {
  //   return !deepCompare(actual, expected.substring(1), comparator);
  // }
  if (_.isArray(actual)) {
    return _.some(actual, function(actualItem) {
      return deepCompare(actualItem, expected, comparator);
    });
  }
  // if (_.isObject(actual)) {
  //   if (_.isObject(expected)) {
  //     return _.every(_.toPlainObject(expected), function(expectedVal, expectedKey) {
  //       if (_.isUndefined(expectedVal)) {
  //         return true;
  //       }
  //       return deepCompare(actual[expectedKey], expectedVal, comparator);
  //     });
  //   } else {
  //     return _.some(actual, function(value, key) {
  //       return deepCompare(value, expected, comparator);
  //     });
  //   }
  // } else {
  //   return comparator(actual, expected);
  // }
}
```

我们现在开发好的对象标准匹配系统看似很灵活，但实际上有点过于宽松了。如果我们有一个对象标准如 `{user: {name: 'Bob'}}`，我们希望它只会匹配到一个拥有属性 `user`，且 `user` 属性中的的 `name` 属性值为 `Bob` 的对象。我们不希望它会匹配到其他层级中出现的 `Bob`：

_test/filter_filter_spec.js_

```js
it('filters with nested objects on the same level only', function() {
  var items = [{ user: 'Bob' },
    { user: { name: 'Bob' } },
    { user: { name: { first: 'Bob', last: 'Fox' } } }
  ];
  var fn = parse('arr | filter:{user: {name: "Bob"}}');
  expect(fn({
    arr: [
      { user: 'Bob' },
      { user: { name: 'Bob' } },
      { user: { name: { first: 'Bob', last: 'Fox' } } }
    ]
  })).toEqual([
    { user: { name: 'Bob' } }
  ]);
});
```

这个测试用例没有通过，这时因为我们当前的标准也会匹配到 `{user: {name: {first: ‘Bob’, last: ‘Fox’}}}`，为什么会这样呢？

原因是一旦我们对期望值和实际值中的 `name` 属性进行遍历，期望值会变成原始字符串 `Bob`。我们已经学习过如果进行对字符串这种原始类型进行匹配，`deepCompare` 会对实际值中所有属性，包括内嵌属性，进行匹配。但在这种情况下，我们不希望进行这种匹配。我们只希望在当前层级上对原始类型进行匹配。

我们需要对 `deepCompare` 进行扩展，让它既可以用来匹配实际值上任意层级的属性，也可以匹配指定层级上的属性。这可以通过一个新参数 `matchAnyProperty` 来进行控制。只有当它为 `true` 时，我们才对实际对象进行递归匹配，否则我们只会进行简单的原始类型级别的比较：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator, matchAnyProperty) {
  // if (_.isString(expected) && _.startsWith(expected, '!')) {
  //   return !deepCompare(actual, expected.substring(1), comparator);
  // }
  // if (_.isArray(actual)) {
  //   return _.some(actual, function(actualItem) {
  //     return deepCompare(actualItem, expected, comparator);
  //   });
  // }
  // if (_.isObject(actual)) {
  //   if (_.isObject(expected)) {
  //     return _.every(
  //       _.toPlainObject(expected),
  //       function(expectedVal, expectedKey) {
  //         if (_.isUndefined(expectedVal)) {
  //           return true;
  //         }
  //         return deepCompare(actual[expectedKey], expectedVal, comparator);
  //       }
  //     );
    } else if (matchAnyProperty) {
      // return _.some(actual, function(value, key) {
      //   return deepCompare(value, expected, comparator);
      // });
    } else {
      return comparator(actual, expected);
      // }
  // } else {
  //   return comparator(actual, expected);
  // }
}
```

现在在判定函数中，我们需要往这个参数传入 `true` 才能恢复我们希望匹配任意属性的默认行为：

_src/filter_fitler.js_

```js
return function predicateFn(item) {
  return deepCompare(item, expression, comparator, true);
};
```

在递归调用 `deepCompare` 时，我们需要把这个参数传递下去，唯一的例外是 `_.every` 函数，我们不需要匹配任意层级的属性：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator, matchAnyProperty) {
  // if (_.isString(expected) && _.startsWith(expected, '!')) {
    return !deepCompare(actual, expected.substring(1),
      comparator, matchAnyProperty);
  // }
  // if (_.isArray(actual)) {
  //   return _.some(actual, function(actualItem) {
      return deepCompare(actualItem, expected,
        comparator, matchAnyProperty);
  //   });
  // }
  // if (_.isObject(actual)) {
  //   if (_.isObject(expected)) {
  //     return _.every(_.toPlainObject(expected), function(expectedVal, expectedKey) {
  //       if (_.isUndefined(expectedVal)) {
  //         return true;
  //       }
        return deepCompare(actual[expectedKey], expectedVal, comparator);
    //   });
    // } else if (matchAnyProperty) {
    //   return _.some(actual, function(value, key) {
        return deepCompare(value, expected,
  //         comparator, matchAnyProperty);
  //     });
  //   } else {
  //     return comparator(actual, expected);
  //   }
  // } else {
  //   return comparator(actual, expected);
  // }
}
```
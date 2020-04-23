### 使用字符串进行过滤
#### Filtering With Strings

对应用开发者来说，每次使用过滤器时都要创建一个判定函数不太方便。这也是为什么过滤器提供了很多方法来满足这个需求。举例来说，你可以给过滤器传入一个字符串，过滤器就会把输入数组中匹配到这个字符串的元素筛选出来：

_test/filter_filter_spec.js_

```js
it('can filter an array of strings with a string', function() {
  var fn = parse('arr | filter:"a"');
  expect(fn({ arr: ['a', 'b', 'a'] })).toEqual(['a', 'a']);
});
```

首先，我们需要检查过滤器接收到的值是什么类型的。如果是函数，我们会将它视作判定函数，但如果是字符串，我需要创建一个判定函数。如果我们无法判断传入的值类型，我们会直接返回输入数组，因为在这种情况下我们根本不知道怎么对这个数组进行过滤：

_src/filter_filter.js_

```js
function filterFilter() {
  // return function(array, filterExpr) {
    var predicateFn;
    if (_.isFunction(filterExpr)) {
      predicateFn = filterExpr;
    } else if (_.isString(filterExpr)) {
      predicateFn = createPredicateFn(filterExpr);
    } else {
      return array;
    }
    return _.filter(array, predicateFn);
  // };
}
```

目前我们只验证输入的字符串是否严格匹配就可以了：

_src/filter_filter.js_

```js
function createPredicateFn(expression) {
  return function predicateFn(item) {
    return item === expression;
  };
}
```

但实际上，过滤器并不需要这么严格。只要字符串中包含了过滤器的输入字符串就可以匹配到了：

_test/filter_filter_spec.js_

```js
it('filters an array of strings with substring matching', function() {
  var fn = parse('arr | filter:"o"');
  expect(fn({ arr: ['quick', 'brown', 'fox'] })).toEqual(['brown', 'fox']);
});
```

我们只需要把判定函数改造成判断是否包含字符串就可以了：

_src/filter_filter.js_

```js
function createPredicateFn(expression) {
  return function predicateFn(item) {
    return item.indexOf(expression) !== -1;
  };
}
```

过滤器在比较时并不会区分大小写：

_test/filter_filter_spec.js_

```js
it('filters an array of strings ignoring case', function() {
  var fn = parse('arr | filter:"o"');
  expect(fn({ arr: ['quick', 'BROWN', 'fox'] })).toEqual(['BROWN', 'fox']);
});
```

我们可以先把表达式和每个数组元素的内容都转为小写，再判断是否匹配：

_src/filter_filter.js_

```js
function createPredicateFn(expression) {
  return function predicateFn(item) {
    var actual = item.toLowerCase();
    var expected = expression.toLowerCase();
    return actual.indexOf(expected) !== -1;
  };
}
```

更有趣的是，如果我们的输入表达式是一个对象数组，使用字符串过滤时会对数组元素中的每个属性进行匹配。这也就是说我们也可以对原始类型以外的值进行过滤：

_test/filter_filter_spec.js_

```js
it('filters an array of objects where any value matches', function() {
  var fn = parse('arr | filter:"o"');
  expect(fn({
    arr: [
      { firstName: 'John', lastName: 'Brown' },
      { firstName: 'Jane', lastName: 'Fox' },
      { firstName: 'Mary', lastName: 'Quick' }
    ]
  })).toEqual([
    { firstName: 'John', lastName: 'Brown' },
    { firstName: 'Jane', lastName: 'Fox' }
  ]);
});
```

在实现这个功能以前，我们先来重构一下代码以便分离问题（separate some concerns）。对两个值进行比较的代码可以拆分出来，放到到一个叫 `comparator` 的内部函数里去：

_src/filter_filter.js_

```js
function createPredicateFn(expression) {

  function comparator(actual, expected) {
    actual = actual.toLowerCase();
    expected = expected.toLowerCase();
    return actual.indexOf(expected) !== -1;
  }
  
  // return function predicateFn(item) {
    return comparator(item, expression);
  // };
}
```

现在，我们可以引入另一个函数，这个函数会接收实际传入值、期望值和一个比较函数（comparator）。这个函数知道如何对传入值进行“深度比较”（deeply compare），如果传入的值是一个对象，它会对对象进行（递归）检索，一旦发现有属性值能匹配到期望值就返回 `true`。如果传入值不是一个对象，就直接使用比较函数进行比较就可以了：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator) {
  if (_.isObject(actual)) {
    return _.some(actual, function(value) {
      return comparator(value, expected);
    });
  } else {
    return comparator(actual, expected);
  }
}
```

如果在判定函数中使用这个新的辅助参数，就能通过我们上面写的测试用例了：

_src/filter_filter.js_

```js
function createPredicateFn(expression) {
  
  // function comparator(actual, expected) {
  //   actual = actual.toLowerCase();
  //   expected = expected.toLowerCase();
  //   return actual.indexOf(expected) !== -1;
  // }

  // return function predicateFn(item) {
    return deepCompare(item, expression, comparator);
  // };
}
```

我们现在会把判定函数的任务拆分到三个函数中去：

- `comparator` 会对两个原始类型的值进行比较
- `deepCompare` 能比较两个原始类型的值，或者对一个包含原始类型值的对象和一个原始类型值进行比较：
- `predicateFn` 会结合使用 `deepCompare` 和 `comparator` 组成最终的过滤器判定函数。

过滤器还能够对任意层级的嵌套对象进行递归匹配：

_test/filter_filter_spec.js_

```js
it('filters an array of objects where a nested value matches', function() {
  var fn = parse('arr | filter:"o"');
  expect(fn({
    arr: [
      { name: { first: 'John', last: 'Brown' } },
      { name: { first: 'Jane', last: 'Fox' } },
      { name: { first: 'Mary', last: 'Quick' } }
    ]
  })).toEqual([
    { name: { first: 'John', last: 'Brown' } },
    { name: { first: 'Jane', last: 'Fox' } }
  ]);
});

```

数组也是一样的。如果你有传入一个多维数组，过滤器能够返回匹配期望值的内嵌数组：

_test/filter_filter_spec.js_

```js
it('filters an array of arrays where a nested value matches', function() {
  var fn = parse('arr | filter:"o"');
  expect(fn({
    arr: [
      [{ name: 'John' }, { name: 'Mary' }],
      [{ name: 'Jane' }]
    ]
  })).toEqual([
    [{ name: 'John' }, { name: 'Mary' }]
  ]);
});
```

我们只需要把 `deepCompare` 变成一个递归函数就可以实现这个功能了。所有在对象（和数组）中的值会被再次传入到 `deepCompare` 中进行处理，而 `comparator` 只会在遇到叶子结点的原始类型值才会被调用：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator) {
  // if (_.isObject(actual)) {
  //   return _.some(actual, function(value) {
      return deepCompare(value, expected, comparator);
  //   });
  // } else {
  //   return comparator(actual, expected);
  // }
}
```
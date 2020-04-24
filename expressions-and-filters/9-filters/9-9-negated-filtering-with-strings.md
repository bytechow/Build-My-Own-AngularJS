### 使用字符串进行反向过滤
#### Negated Filtering With Strings

在某些特定情况下，过滤不匹配条件的数组元素比过滤匹配条件的更有用。我们只需要在过滤字符串前加一个 `!` 前缀就可以了：

_test/filter_filter_spec.js_

```js
it('allows negating string filter', function() {
  var fn = parse('arr | filter:"!o"');
  expect(fn({ arr: ['quick', 'brown', 'fox'] })).toEqual(['quick']);
});
```

要实现这个功能也很简单。在 `deepCompare` 传入的、作为匹配条件的字符串是以 `!` 开头的话，我们会将去掉了 `!` 的字符串作为参数再次调用 `deepCompare` ，然后对结果进行反转：

_src/filter_filter.js_

```js
function deepCompare(actual, expected, comparator) {
  if (_.isString(expected) && _.startsWith(expected, '!')) {
    return !deepCompare(actual, expected.substring(1), comparator);
  }
  // if (_.isObject(actual)) {
  //   return _.some(actual, function(value, key) {
  //     return deepCompare(value, expected, comparator);
  //   });
  // } else {
  //   return comparator(actual, expected);
  // }
}
```
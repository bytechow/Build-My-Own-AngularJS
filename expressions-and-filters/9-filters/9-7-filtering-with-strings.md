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
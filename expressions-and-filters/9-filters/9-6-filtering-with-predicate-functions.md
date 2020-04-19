### 使用判定函数进行过滤
#### Filtering With Predicate Functions

就实现而言，使用 filter 过滤器最简单的方法就是提供一个判定函数。过滤器会利用这个函数返回一个新的数组，其中只包含经过判定函数判断结果为真的元素：

_test/filter_filter_spec.js_

```js
it('can filter an array with a predicate function', function() {
  var fn = parse('[1, 2, 3, 4] | filter:isOdd');
  var scope = {
    isOdd: function(n) {
      return n % 2 !== 0;
    }
  };
  expect(fn(scope)).toEqual([1, 3]);
});
```

现在我们需要在 spec 文件中引入 `parse`：

_test/filter_filter_spec.js_

```js
// 'use strict';

var parse = require('../src/parse');
// var filter = require('../src/filter').filter;
```

之所以实现起来如此简单，是因为我们可以把这个任务委托给 LoDash 的 [filter 函数](https://lodash.com/docs/4.17.15?filter#filter)。它会接收一个数组和一个判定函数，返回一个新数组：

_src/filter_filter.js_

```js
// 'use strict';

var _ = require('lodash');

// function filterFilter() {
  return function(array, filterExpr) {
    return _.filter(array, filterExpr);
//   };
// }

// module.exports = filterFilter;
```
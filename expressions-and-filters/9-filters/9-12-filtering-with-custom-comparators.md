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
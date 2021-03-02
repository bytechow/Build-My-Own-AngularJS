### 处理带有 length 属性的对象
#### Dealing with Objects that Have A length

到目前为止，我们已经处理了绝大多数集合了，但还要处理一种特殊的对象。

回想一下，我们判断一个对象是否是类数组对象是通过检查它是否有数字类型的 `length` 属性值。那我们又该如何处理下面这个对象呢？

```js
{
  length: 42,
  otherKey: 'abc'
}
```

这并不是一个类数组对象，它只是刚好也有一个 `length` 属性而已。应用程序还是可能会存在这种对象的，所以我们也要对它进行处理。

我们新增一个单元测试，测试一个含有 `length` 属性的对象发生变化时是否会被检测到：

_test/scope_spec.js_

```js
it('does not consider any object with a length property an array', function() {
  scope.obj = { length: 42, otherKey: 'abc' };
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$digest();
  
  scope.obj.newKey = 'def';
  scope.$digest();
  
  expect(scope.counter).toBe(2);
});
```

运行这个测试时，虽然对象属性发生了改变，但 listener 函数并没有被调用。那是因为这个对象的 `length` 属性会让程序误以为它是一个数组，从而采用针对数组的变化侦测方式，这种方式并不能侦测到对象属性新增这种变化。

要修复这个问题也非常简单。我们要更改判断条件，不再认为只要含有 `length` 属性的对象就是类数组对象，而是判断 `length` 属性是否是一个数字类型，同时要求 `length` 属性的值减 1 的索引位置上也应该存在一个属性，通过这样的方式来缩窄范围。举个例子，如果一个对象含有 `length` 属性，属性值为 42，如果这个对象还有一个键为 `41` 的属性在对象上，那它就是数组或类数组对象。

但这只能用在 `length` 属性不为 0 的情况上，所以我们需要也要兼容长度为 0 的情况：

_src/scope.js_

```js
function isArrayLike(obj) {
  if (_.isNull(obj) || _.isUndefined(obj)) {
    return false;
  }
  var length = obj.length;
  return length === 0 ||
    (_.isNumber(length) && length > 0 && (length - 1) in obj);
}
```

这样我们的测试就能通过了，而且也适用于大多数对象。这个检查并不万无一失的，但实际上已经是我们能做到的最好情况了。
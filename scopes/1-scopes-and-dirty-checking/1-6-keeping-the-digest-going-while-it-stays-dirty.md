### 尚有脏值存在时继续运行 digest（Keeping The Digest Going While It Stays Dirty）

虽然我们已经实现了脏值检测的核心了，但我们离完成还远着呢。举例来说，我们就还没能支持这样一个典型场景：listener 函数可能也会改变作用域上的属性。如果出现了这种情况，并且还有另一个 watcher 在侦听该属性时，那么在当前 digest 运行过程中它无法察觉到这个属性是发生了变化的：

_test/scope_spec.js_

```js
it('triggers chained watchers in the same digest', function() {
  scope.name = 'Jane';
  
  scope.$watch(
    function(scope) { return scope.nameUpper; },
    function(newValue, oldValue, scope) {
      if (newValue) {
        scope.initial = newValue.substring(0, 1) + '.';
      }
    }
  );

  scope.$watch(
    function(scope) { return scope.name; },
    function(newValue, oldValue, scope) {
      if (newValue) {
        scope.nameUpper = newValue.toUpperCase();
      }
    }
  );
  
  scope.$digest();
  expect(scope.initial).toBe('J.');

  scope.name = 'Bob';
  scope.$digest();
  expect(scope.initial).toBe('B.');
});
```

我们在作用域里面有两个 watcher：一个会侦听 `nameUpper` 属性，它会根据这个属性来生成`initial`属性，而另一个侦听的是 `name` 属性，并根据这个属性生成赋给 `nameUpper` 属性的值。我们希望产生的效果是，在同一个 digest 阶段中，当作用域上的 `name` 发生变化时，`nameUpper` 和 `initial` 的属性值会相应发生改变。但目前的情况并不是这样的。
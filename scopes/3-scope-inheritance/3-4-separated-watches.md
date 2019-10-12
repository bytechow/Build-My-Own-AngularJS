### 分离 watcher
#### Separated Watches

我们已经知道可以往子作用域上添加 watcher，这是因为子作用域能够继承父作用域上的所有方法，包括 `$watch` 和 `$digest`。但这些 watcher 实际上会被保存在哪里、又会在哪个作用域的基础上进行调用呢？

在目前的代码实现中，所有的 watcher 实际上都被保存到根作用域上了。这是因为我们是在 `Scope`，也就是根作用域的构造函数中定义 `$$watchers` 数组的。当任何一个子作用域访问 `$$watchers` 数组（或其他任何在这个构造函数上初始化的属性）时，它们通过原型链访问到的只是一个根作用域的拷贝。

这里会产生一个严重的影响：无论我们会对哪个作用域调用 `$digest` 方法，都会导致整个作用域继承链上的所有 watcher 被执行。这是因为我们只有一个存放 watcher 的属性：这个数组是放在根作用域上的。这并不是我们想要的效果。

我们希望当我们调用 `$digest` 只会执行当前作用域及其子作用域上的 watch 函数。而不像现在那样，会把父作用域及其他子作用域上的 watch 函数也执行掉：

```js
it('does not digest its parent(s)', function() {
  var parent = new Scope();
  var child = parent.$new();

  parent.aValue = 'abc';
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  
  child.$digest();
  expect(child.aValueWas).toBeUndefined();
});
```
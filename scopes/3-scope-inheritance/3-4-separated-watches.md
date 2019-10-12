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

这个测试失败的原因是我们调用了 `child.$digest()`，我们实际上是执行了 `parent` 上的 watch。让我们来修复这个问题。

其中的诀窍就是在每个子作用域都指派一个自己的 `$$watchers` 数组：

_src/scope.js_

```js
Scope.prototype.$new = function() {
  // var ChildScope = function() { };
  // ChildScope.prototype = this;
  // var child = new ChildScope();
  child.$$watchers = [];
  // return child;
};
```

你可能也注意到了，我们在这里使用了上一节提到的属性屏蔽。每个在子作用域上定义的 `$$watchers` 数组属性都会屏蔽父作用域上的同名属性。整个作用域链所有层次的作用域都有属于自己的 watcher。现在，当我们在某个作用域上调用 `$digest`，只有这个作用域上的 watcher 会被调用。
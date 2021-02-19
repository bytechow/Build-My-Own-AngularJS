### 分离侦听器

#### Separated Watches

上面我们已经知道可以往子作用域上添加 watcher，这是因为子作用域继承了父作用域上的所有方法，包括 `$watch` 和 `$digest`。但这些 watcher 实际上会被保存在哪里呢？又是在哪个作用域的基础上进行调用的呢？

目前所有的 watcher 都保存到根作用域上了。这是因为我们是在 `Scope` 这个根作用域的构造函数上定义 `$$watchers` 数组的。当子作用域访问 `$$watchers`（或构造函数中初始化的任何其他属性）时，它们会通过原型链获取这个属性在根作用域的副本。

这会产生一个严重影响：无论我们在哪个作用域上调用 `$digest` 方法，都会执行作用域继承链上的所有 watcher，毕竟我们只在根作用域上存放了 watcher 数组而已。这并不是我们想要的效果。

我们希望在调用 `$digest` 时，只执行当前作用域及其子作用域上的 watch 函数。但现在我们把父作用域上以及其他无关子作用域上的 watch 函数也执行了：

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

这个测试没有通过，原因是当我们调用 `child.$digest()` 时，实际上是执行了 `parent` 上的 watch 函数。下面，我们来修复这个问题。

解决方案就是为每个子作用域都初始化一个 `$$watchers` 数组：

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

你可能也注意到了，我们在这里利用了上一节介绍过的属性屏蔽特性。每个在子作用域上定义的 `$$watchers` 数组属性都会屏蔽父作用域上的同名属性。最终，作用域链所有层次的作用域都会有属于自己的 watcher。现在，当我们在某个作用域上调用 `$digest` 方法，只会执行这个作用域上的 watcher。


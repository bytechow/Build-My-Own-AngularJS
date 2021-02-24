### 销毁作用域

#### Destroying Scopes

在一个典型的 Angular 应用的生命周期中，展示给用户看的视图和数据不同，页面上的元素也会跟着增减变化。这也意味着在应用的生命周期中，随着控制器和指令作用域的添加和删除，作用域的树结构会扩展和收缩。

现在我们已经可以创建子作用域了，但还没有移除它们的机制。从性能方面考虑，一个不断扩展的作用域树结构会带来麻烦——尤其是它里面附带的所有 watcher！显然，我们需要一种销毁作用域的方式。

销毁一个作用域意味着会把这个作用域的所有 watcher 都移除掉，同时也会从它的父作用域上的 `$$children` 中移除掉对这个作用域的引用。由于已经没有任何地方对这个作用域进行引用，它将会在之后的某个时间点被 JavaScript 环境中的垃圾收集器回收掉。（当然，这只会发生在没有任何外部代码引用到这个作用域或它的 watcher 的情况下。）

我们会在一个名为 `$destroy` 的 Scope 方法中实现这个销毁功能。只要调用这个函数，就会销毁对应的作用域：

_test/scope\_spec.js_

```js
it('is no longer digested when $destroy has been called', function() {
  var parent = new Scope();
  var child = parent.$new();

  child.aValue = [1, 2, 3];
  child.counter = 0;
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    },
    true
  );

  parent.$digest();
  expect(child.counter).toBe(1);

  child.aValue.push(4);
  parent.$digest();
  expect(child.counter).toBe(2);

  child.$destroy();
  child.aValue.push(5);
  parent.$digest();
  expect(child.counter).toBe(2);
});
```

在 `$destroy` 方法中，我们需要有一个引用指向当前作用域的父作用域。但现在还没有，我们会在 `$new` 函数中增加一个属性（指向父作用域）。每个子作用域被创建时，它的直接（或替代）父作用域会被赋值给 `$parent` 属性：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated, parent) {
  // var child;
  // parent = parent || this;
  // if (isolated) {
  //   child = new Scope();
  //   child.$root = parent.$root;
  //   child.$$asyncQueue = parent.$$asyncQueue;
  //   child.$$postDigestQueue = parent.$$postDigestQueue;
  //   child.$$applyAsyncQueue = parent.$$applyAsyncQueue;
  // } else {
  //   var ChildScope = function() {};
  //   ChildScope.prototype = this;
  //   child = new ChildScope();
  // }
  // parent.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  child.$parent = parent;
  // return child;
};
```

> 注意 `$parent` 只使用了一个美元符号，而不是两个。这就意味着 Angular 开发团队认为可以在应用程序代码中引用这个属性。但是这样的做法通常被认为是一种反面模式，因为会带来作用域之间的强耦合。

现在我们可以开始实现 `$destroy` 了。这个函数会从父作用域中找到当前作用域的位置，然后移除当前作用域——只要这个作用域不是根作用域而且是有父作用域。同时，还会把作用域上的 watcher 都移除掉：

_src/scope.js_

```js
Scope.prototype.$destroy = function() {
  if (this.$parent) {
    var siblings = this.$parent.$$children;
    var indexOfThis = siblings.indexOf(this);
    if (indexOfThis >= 0) {
      siblings.splice(indexOfThis, 1);
    }
  }
  this.$$watchers = null;
};
```




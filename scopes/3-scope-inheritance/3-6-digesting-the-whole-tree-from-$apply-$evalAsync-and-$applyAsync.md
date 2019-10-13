### 调用 `$apply`，`$evalAsync` 和 `$applyAsync` 时会对整个树结构进行 digest
#### Digesting The Whole Tree from $apply, $evalAsync, and $applyAsync

正如我们在上面的章节中看到的那样，`$digest` 只会从当前作用域开始往下执行。但对于 `$apply` 来说，情况就不一样了。当我们在 Angular 中调用 `$apply`，它会从整个作用域树结构的最顶端开始执行 digest。下面这个单元测试就能说明目前我们还没实现这样的效果：

_test/scope_spec.js_

```js
it('digests from root on $apply', function() {
  var parent = new Scope();
  var child = parent.$new();
  var child2 = child.$new();

  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  child2.$apply(function(scope) {});
  
  expect(parent.counter).toBe(1);
});
```

我们能看到，当我们在（孙）子元素上调用 `$apply` 时，并未能触发它祖父元素上的 watcher。

要实现这个效果，我们首先需要在作用域上保存它们的根元素的引用，这样才能在根元素上触发 digest。我们也可以沿着原型链找到根作用域，但显式地在作用域上保存一个 `$root` 属性会直接得多。我们可以在根作用域的构造函数上设置这个变量：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  // this.$$applyAsyncQueue = [];
  // this.$$applyAsyncId = null;
  // this.$$postDigestQueue = [];
  this.$root = this;
  // this.$$children = [];
  // this.$$phase = null;
}
```

这样的话，利用原型继承链的特性，树结构里面所有的作用域就都能访问到这个 `$root` 变量了。

我们仍然需要在 `$apply` 中进行修改，但也非常简单。我们在根作用域上调用 `$digest`，而不是在当前作用域上调用：

_src/scope.js_

```js
Scope.prototype.$apply = function(expr) {
  try {
    this.$beginPhase('$apply');
    return this.$eval(expr);
  } finally {
    this.$clearPhase();
    this.$root.$digest();
  }
};
```

注意，这里我们还是会在当前作用域的语境下运行传入的函数，而不是在根作用域上，通过调用 `this` 上的 `$eval` 方法。我们只是希望 digest 能从根作用域开始一直运行下来而已。
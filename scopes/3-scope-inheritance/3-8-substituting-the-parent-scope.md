### 替换父作用域
#### Substituting The Parent Scope

在某些情况下，在保持正常的原型继承链的前提下，指定其他作用域作为新作用域的父作用域是非常有用的：

_test/scope_spec.js_

```js
it('can take some other scope as the parent', function() {
  var prototypeParent = new Scope();
  var hierarchyParent = new Scope();
  var child = prototypeParent.$new(false, hierarchyParent);

  prototypeParent.a = 42;
  expect(child.a).toBe(42);
  
  child.counter = 0;
  child.$watch(function(scope) {
    scope.counter++;
  });
  
  prototypeParent.$digest();
  expect(child.counter).toBe(0);
  
  hierarchyParent.$digest();
  expect(child.counter).toBe(2);
});
```

这里我们创建了两个“父”作用域，然后再创建一个子作用域。其中一个父作用域是新作用域原来的、原型链上的父作用域。而另一个是“树结构上”的父作用域，会作为 `$new` 的第二个参数传入。

我们会测试子作用域与原型链上的父作用域之间的继承关系是否正常，同时也会验证原型链上的父作用域启动 digest 时是否 **不会** 执行子作用域上的 watcher。相反，当 _树结构上_ 的父作用域运行 digest 时就会执行子作用域上的 watcher 了。

我们会在 `$new` 函数中加入可选的第二个参数，它默认指向当前作用域 `this`。然后我们会在这个作用域存储子作用域的数组中加入这个新的子作用域：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated, parent) {
  // var child;
  parent = parent || this;
  if (isolated) {
    // child = new Scope();
    child.$root = parent.$root;
    child.$$asyncQueue = parent.$$asyncQueue;
    child.$$postDigestQueue = parent.$$postDigestQueue;
    child.$$applyAsyncQueue = parent.$$applyAsyncQueue;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  }
  parent.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

> 我们也会使用 `parent` 来访问孤立作用域中各个不同的队列。这些队列会被所有作用域共享，所以其实无论用 `this` 还是 `parent` 去访问都是一样的，但为了清晰起见，我们还是统一使用后者。关键还是在于我们是往哪个作用域 `$$children` 去添加这个新作用域。

从这个特性中，我们也能够觉察出原型链继承和树结构继承的细微差别。在大多数情况下这种差别很小，并不值得我们耗费脑力去跟踪这两个存在细微差别的作用域层次关系。但到本书后面，我们要开发指令 transclusion 的时候，就能看到这样区分的好处。
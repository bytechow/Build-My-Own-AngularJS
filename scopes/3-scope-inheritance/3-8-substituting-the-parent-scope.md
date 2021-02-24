### 替换父作用域

#### Substituting The Parent Scope

有时，在保持原型继承正常运作的前提下，指定其他作用域作为新作用域的父级作用域是挺有用的：

_test/scope\_spec.js_

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

这里我们创建了两个“父”作用域，然后再创建一个子作用域。其中一个是新作用域在原型链上的父作用域。而另一个是“树结构上”的父作用域，后者会作为 `$new` 的第二个参数传入。

我们先测试子作用域与原型链上的父作用域之间的继承关系是否正常，然后验证原型链上的父作用域启动 digest 时是否 **不会** 执行子作用域上的 watcher。相反，当 _树结构上_ 的父作用域运行 digest 时就会执行子作用域上的 watcher 了。

我们在 `$new` 函数加入可选的第二个参数，它默认指向当前作用域 `this`。然后把这个新的子作用域加入到子作用域数组中去：

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

> 同时我们也会使用 `parent` 来访问隔离作用域中各个队列。由于这些队列会被所有作用域共享（指向的都是根作用域的队列），我们无论是用 `this` 还是用 `parent` 访问都可以，但为了清晰起见，我们还是统一使用后者访问。我们往哪个作用域的 `$$children` 数组添加这个新作用域才是关键。

实现这个功能之后，原型链继承和树层次继承会产生细微差别。在大多数情况下，这种差别可以说是没有价值，并不值得我们耗费资源去缓存两个只存在细微差别的作用域层次关系。但后面我们开发指令 transclusion 的时候，就会知道有时这种差别也是有用的。


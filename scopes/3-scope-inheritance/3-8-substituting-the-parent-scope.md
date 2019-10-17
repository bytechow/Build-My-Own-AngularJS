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

这里我们创建了两个“父”作用域，然后再创建一个子作用域。其中一个父作用域是新作用域原来的、原型链上的父作用域。而另一个是“层次结构上”的父作用域，会作为 `$new` 的第二个参数传入。

我们会测试子作用域与原型链上的父作用域之间的继承关系是否正常，同时也会验证原型链上的父作用域启动 digest 时是否<u>不会</u>执行子作用域上的 watcher。相反，当 _层次结构上_ 的父作用域运行 digest 时就会执行子作用域上的 watcher 了。
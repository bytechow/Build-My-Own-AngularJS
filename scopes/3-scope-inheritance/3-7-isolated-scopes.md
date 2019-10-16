### 孤立的作用域
#### Isolated Scopes

我们已经看到了父子作用域在原型继承链上的联系是多么的紧密。无论父作用域上有什么属性，子作用域都可以访问到。如果这个属性是一个对象或者数组，子作用域还可以修改它的内容。

但有时候我们并不需要这么紧密的联系。有时，如果能有一个作用域本身是作用域树的一部分但没有权限访问父作用域数据的话，是很方便的。这就是所谓的_孤立的作用域_（isolated scope）。

孤立作用域背后的思想是很简单的：我们会像之前一样创建一个作用域，它会作为作用域树的一部分，但我们不会让它在原型链上对它的父作用域进行继承。它会从父作用域的原型链结构中切割出来，或者说是被孤立起来。

我们可以通过传递一个布尔值参数给 `$new` 函数来创建一个孤立作用域。当这个参数值为 `true` 时，创建的作用域就会是孤立的。当只为 `false`（或者是被忽略了，或者是 `undefined`），就会使用原型继承。当作用域是孤立的，它就无法访问到父作用域上的数据：

_test/scope_spec.js_

```js
it('does not have access to parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';
  
  expect(child.aValue).toBeUndefined();
});
```

而且由于我们已经无法访问到父作用域上的属性，我们自然就没办法对这些属性进行侦听了：

```js
it('cannot watch parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  
  child.$digest();
  expect(child.aValueWas).toBeUndefined();
});
```

我们会在 `$new` 函数中设置孤立的作用域。根据传入的布尔值属性来决定是创建跟以往一样的子作用域，还是使用 `Scope` 构造函数来创建一个独立的作用域。这两种方式创建的新作用域都会被添加到当前作用域的子作用域数组（children）中去：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

> 如果你在 Angular 指令中用过孤立作用域，你会知道其实孤立作用域通常不会跟它的父作用域完全割裂开来的。相反，我们会明确定义出需要从父作用域中获取的属性映射。
> 
> 然而，作用域中并没有加入这个机制。这个机制是指令功能的一部分。后面我们讲到指令作用域链接（directive scope linking）的时候会再来讨论这个知识点。


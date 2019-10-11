### 属性屏蔽
#### Attribute Shadowing

作用域继承还有一个特性可能会绊倒 Angular 新手。因为这是使用 JavaScript 原型链后的直接结果，我们有必要了解一下。

在我们现有的测试用例中可以清晰地看到，当要从一个作用域_读取_（read）一个属性时，如果当前作用域中没找到这个属性，会从它的父作用域上查找，依此类推，继续在原型链上查找。其次，如果我们在作用域指派（assign）一个属性后，那这个属性只能被当前作用域及其子作用域访问到，而不包括当前作用域的父作用域。

关键是，这条规则在我们在子作用域上重用一个属性名的情况依然适用：

_test/scope_spec.js_

```js
it('shadows a parents property with the same name', function() {

  var parent = new Scope();
  var child = parent.$new();
  
  parent.name = 'Joe';
  child.name = 'Jill';
  
  expect(child.name).toBe('Jill');
  expect(parent.name).toBe('Joe');
});
```

当我们把一个已经在父作用域上存在的（属性名称对应的）属性放到子作用域中进行指派时，并不会改变父作用域上的同名属性。事实是，现在在作用域原型链上有两个不同的属性，这两个属性都叫做 `name` 而已。这种现象一般被称为_屏蔽_（shadowing）：从子作用域的角度看，父作用域上的 `name` 属性被子作用域上的 `name` 属性屏蔽掉了。

这经常成为大家困惑的来源，当然，也有很多真的需要更改父作用域上状态的情况。解决这个问题的一种常用方法是将属性封装到一个对象中。因为即使在这种情况下，对象中的内容依然是可以被更改（就像上一章节中关于数组操作的例子一样）：

_test/scope_spec.js_

```js
it('does not shadow members of parent scopes attributes', function() {
  var parent = new Scope();
  var child = parent.$new();
 
  parent.user = {name: 'Joe'};
  child.user.name = 'Jill';
 
  expect(child.user.name).toBe('Jill');
  expect(parent.user.name).toBe('Jill');
});
```

在这个例子中，我们可以成功修改父作用域上的属性的原因是，我们并没有对子作用域指派（assign）任何的属性。我们只是从作用域中读取 `user` 对象属性，并对这个对象的属性进行指派而已。两个作用域都持有访问同一个 `user` 对象的引用，而这个 `user` 对象只是一个普通的 JavaScript 对象，与作用域继承机制无关。

> 这种模式也被称为 _Dot Rule_，这与我们在改变作用域时用的表达式中用于访问属性的点运算符的数量有关。正如 [Misko Hevery 说的](https://www.youtube.com/watch?feature=player_detailpage&v=ZhfUv0spHCY#t=1758s)那样，“无论你在何时使用 ngModel，总会在某处有一个点运算符。如果一个点运算符都没有，你就用错了。”
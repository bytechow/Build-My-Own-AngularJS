### 属性屏蔽
#### Attribute Shadowing

作用域继承中的一个常见问题是属性屏蔽。虽然这是使用 JavaScript 原型链后的直接结果，但有必要了解一下。

从我们现有的测试用例可以清楚地看到，当要_读取_（read）作用域上的一个属性时，程序会在原型链上查找，如果在当前作用域中没找到这个属性，会继续在它的父作用域上查找，依此类推。另外，如果我们在作用域对一个属性进行赋值以后，这个属性能被当前作用域及其子作用域访问到，当前作用域的父作用域则不行。

关键是，这条规则在子作用域使用同名属性的情况下依然适用：

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

当我们对子作用域上的一个已经存在于父作用域上的属性进行赋值时，并不会影响父作用域上的同名属性。实际上，现在在作用域原型链上有两个不同的属性，只不过两个属性都恰好是叫做 `name` 而已。这种现象一般被称为_屏蔽_（shadowing）：从子作用域的角度来看，是父作用域上的 `name` 属性被子作用域上的 `name` 属性屏蔽掉了。

这经常会让人困惑。当然也有在父作用域中改变状态的真实用例。解决这个问题的常用方法是将属性封装到一个对象中。因为即使在这种情况下，对象中的内容依然是可以被更改（就像上一节中的数组操作示例一样）：

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

在这个例子中，我们就可以成功修改父作用域上的属性了，原因是我们并没有对子作用域上的任何属性进行赋值，只是从作用域中读取 `user` 对象属性，并对这个对象上的属性进行赋值而已。这两个作用域都在引用同一个 `user` 对象，而这个 `user` 对象也只是一个普通的 JavaScript 对象，与作用域继承机制并无关系。

> 这种模式也被称为_“点运算规则”_（Dot Rule），指的是在表达式中对作用域上的属性进行操作时使用的点运算符。正如 [Misko Hevery 所说的](https://www.youtube.com/watch?feature=player_detailpage&v=ZhfUv0spHCY#t=1758s)，“无论你在什么时候使用 ngModel，同时肯定会出现一个点运算符。如果一个点运算符都没有，那就说明你用错了。”
### 属性屏蔽

#### Attribute Shadowing

作用域继承中的一个常见问题是属性屏蔽。要使用 JavaScript 原型链就无法避免这个问题，所以有必要了解一下。

从我们现有的测试用例可以清楚地看到，当要_读取_（read）作用域上的一个属性时，程序会在原型链上查找，如果在当前作用域中没找到这个属性，会继续在它的父作用域上查找，依此类推。另外，如果我们在作用域中对一个属性进行赋值以后，这个属性能被当前作用域及当前作用域的子作用域访问到，但父作用域无法访问到。

关键是，这条规则在父子作用域存在同名属性的情况下依然适用：

_test/scope\_spec.js_

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

当我们对子作用域上的一个已经存在于父作用域上的属性进行赋值时，并不会影响父作用域上的同名属性。实际上，现在在作用域原型链上有两个不同的属性，只不过两个属性都恰好叫做 `name` 而已。这种现象被称为_屏蔽_（shadowing）：从子作用域的角度来看，是父作用域上的 `name` 属性被子作用域上的 `name` 属性屏蔽掉了。

这是特性经常会让人感到困惑。当然，实际上也会有要改变父作用域状态的情况。解决这种问题的常用方法是将属性封装到一个对象中。因为即使被屏蔽了，对象中的内容依然是可以被更改（就像上一节中的数组操作示例一样）：

_test/scope\_spec.js_

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

像上面这个例子就可以修改父作用域上的属性，因为我们并没有在子作用域中对属性进行赋值，只是从作用域中读取 `user` 对象属性并对这个属性进行赋值而已。这两个作用域引用的是同一个 `user` 对象，这个对象只是一个普通的 JavaScript 对象，与作用域继承机制无关。

> 这种模式也被称为_“点运算规则”_（Dot Rule），指的是在表达式中对作用域上的属性进行操作时使用的点运算符。正如 [Misko Hevery 所说的](https://www.youtube.com/watch?feature=player_detailpage&v=ZhfUv0spHCY#t=1758s)，“无论你在什么时候使用 ngModel，都要有一个点运算符。如果一个点运算符都没有，那说明你用错了。”




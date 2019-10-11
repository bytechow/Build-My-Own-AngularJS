### 属性屏蔽
#### Attribute Shadowing

作用域继承还有一个特性可能会绊倒 Angular 新手。因为这是使用 JavaScript 原型链后的直接结果，值得我们去深入了解一下。

在我们现有的测试用例中可以清晰地看到，当要从一个作用域_读取_（read）一个属性时，如果当前作用域中没找到这个属性，会从它的父作用域上查找，依此类推，继续在原型链上查找。其次，如果我们在作用域对一个属性进行赋值（assign）后，那这个属性只能被当前作用域及其子作用域访问到，而不包括当前作用域的父作用域。

> 译者注：这段中使用的词语——**赋值（assign）**并不准确，应该改为**定义（define）**

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

当我们把一个已经在父作用域上存在的属性再次在子作用域上定义时，
### 创建一个子作用域
#### Making A Child Scope

虽然我们要创建多少个根作用域都可以，但一般情况下，我们只会对一个已经存在的作用域创建子作用域（或者让 Angular 为我们创建）。我们可以通过对已经存在的作用域调用 `$new` 函数来创建子作用域。

下面我们来试着开发 `$new`。在开始之前，我们要在 `test/scope_spec.js` 中添加一个内嵌的 `describe` 代码块，在这个代码块中我们会加入与作用域继承相关的单元测试，结构如下：

_test_

```js
describe('Scope', function() {
  
  describe('digest', function() {
  
    // Tests from the previous chapter...
  
  });
  
  describe('$watchGroup', function() {
  
    // Tests from the previous chapter...
  
  });
  
  describe('inheritance', function() {
  
    // Tests for this chapter
  
  }); 

});
```

子作用域的第一个特点就是它可以共享父作用域上的属性：

_test/scope_spec.js_

```js
it("inherits the parent's properties", function() {
  var parent = new Scope();
  parent.aValue = [1, 2, 3];

  var child = parent.$new();
  
  expect(child.aValue).toEqual([1, 2, 3]);
});
```

但反过来就不一样了。父作用域无法访问子作用域上定义的属性：

_test/scope_spec.js_

```js
it('does not cause a parent to inherit its properties', function() {
  var parent = new Scope();

  var child = parent.$new();
  child.aValue = [1, 2, 3];
  
  expect(parent.aValue).toBeUndefined();
});
```

作用域属性的共享规则跟定义属性的时间点是无关的。父作用域上定义属性后，它的所有子子作用域都可以访问到这个属性：

_test/scope_spec.js_

```js
it('inherits the parents properties whenever they are defined', function() {
  var parent = new Scope();
  var child = parent.$new();

  parent.aValue = [1, 2, 3];
  
  expect(child.aValue).toEqual([1, 2, 3]);
});
```

我们也可以通过子作用域对父作用域上的属性进行修改，因为两个作用域指向的值实际上是相同的：

_test/scope_spec.js_

```js
it('can manipulate a parent scopes property', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = [1, 2, 3];

  child.aValue.push(4);

  expect(child.aValue).toEqual([1, 2, 3, 4]);
  expect(parent.aValue).toEqual([1, 2, 3, 4]);
});
```

由此还可以看出，我们可以在子作用域中对父作用域上的属性进行侦听：

_test/scope_spec.js_

```js
it('can watch a property in the parent', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = [1, 2, 3];
  child.counter = 0;

  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    },
    true
  );
  
  child.$digest();
  expect(child.counter).toBe(1);
  
  parent.aValue.push(4);
  child.$digest();
  expect(child.counter).toBe(2);
});
```

> 你可能发现了，子作用域也是可以调用 `$watch` 函数的，而这个 `$watch` 函数是定义在 `Scope.prototype` 上的。这与我们自定义属性使用的继承机制是一样的：因为父作用域继承自 `Scope.prototype`，而子作用域继承自父作用域，`Scope.prototype` 上定义的一切就能被每一个作用域访问到了！

最后，上面提到的所有特性在作用域继承的任意深度上都适用：

_test/scope_spec.js_

```js
it('can be nested at any depth', function() {
  var a = new Scope();
  var aa = a.$new();
  var aaa = aa.$new();
  var aab = aa.$new();
  var ab = a.$new();
  var abb = ab.$new();

  a.value = 1;
  
  expect(aa.value).toBe(1);
  expect(aaa.value).toBe(1);
  expect(aab.value).toBe(1);
  expect(ab.value).toBe(1);
  expect(abb.value).toBe(1);
  
  ab.anotherValue = 2;
  
  expect(abb.anotherValue).toBe(2);
  expect(aa.anotherValue).toBeUndefined();
  expect(aaa.anotherValue).toBeUndefined();
});
```

虽然目前我们对作用域继承进行了很多定义，但它实现起来其实很简单直接。我们只需要利用 JavaScript 对象继承就可以了，因为 Angular 的作用域是严格按照 JavaScript 本身的工作方式进行建模的。实质上，当你创建子作用域时，它的父作用域就自动成为它的原型了。

> 我们不会花费时间在介绍 JavaScript 中的原型概念上。如果你想复习一下的话，DailyJS 上的两篇文章会很适合你，[prototype](http://dailyjs.com/2012/05/20/js101-prototype/) 和 [inheritance](http://dailyjs.com/2012/05/27/js101-prototype-chains/)。

我们在 `Scope`  构造函数中创建 `$new` 函数。这个函数能够为当前作用域创建一个子作用域，并返回这个子作用域：

```js
Scope.prototype.$new = function() {
  var ChildScope = function() {};
  ChildScope.prototype = this;
  var child = new ChildScope();
  return child;
};
```

在这个函数中，我们首先创建子作用域的构造函数，然后把它赋值给一个叫 `ChildScope` 的变量上。这个构造函数并不需要加入任何代码，所以只要一个空函数就可以了。然后我们会把 `Scope` 设置为 `ChildScope` 的原型。最后使用 `ChildScope` 构造函数创建一个对象并返回这个对象即可。

这个函数没多少代码，但已经能让我们在本节中的所有测试用例都通过了！

> 我们还可以使用 ES5 的简写函数（shorthand function）`Object.create()` 来构建子作用域。
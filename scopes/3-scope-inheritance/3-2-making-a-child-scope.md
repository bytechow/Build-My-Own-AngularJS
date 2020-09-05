### 创建一个子作用域

#### Making A Child Scope

虽然你可以创建任意多个根作用域，但一般情况下，我们只会在一个已存在的作用域下创建子作用域（或者让 Angular 为我们创建）。我们可以通过调用已存在作用域的 `$new` 方法来创建一个子作用域。

下面我们要使用测试驱动的模式来开发 `$new`。在开始之前，我们需要先在 `test/scope_spec.js` 中添加一个内嵌的 `describe` 代码块，在这个代码块中我们会加入与作用域继承相关的单元测试，结构如下：

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

_test/scope\_spec.js_

```js
it("inherits the parent's properties", function() {
  var parent = new Scope();
  parent.aValue = [1, 2, 3];

  var child = parent.$new();

  expect(child.aValue).toEqual([1, 2, 3]);
});
```

但反过来，父作用域是无法访问子作用域上定义的属性的：

_test/scope\_spec.js_

```js
it('does not cause a parent to inherit its properties', function() {
  var parent = new Scope();

  var child = parent.$new();
  child.aValue = [1, 2, 3];

  expect(parent.aValue).toBeUndefined();
});
```

父子作用域对属性的共享与属性何时定义无关。在父作用域上定义属性时，父作用域现存的子作用域同样可以访问到这个属性：

_test/scope\_spec.js_

```js
it('inherits the parents properties whenever they are defined', function() {
  var parent = new Scope();
  var child = parent.$new();

  parent.aValue = [1, 2, 3];

  expect(child.aValue).toEqual([1, 2, 3]);
});
```

我们也可以在子作用域上对父作用域的属性进行修改，因为两个作用域的同名属性实际上指向同一个值：

_test/scope\_spec.js_

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

由此还可以看出，子作用域可以对父作用域上的属性进行侦听：

_test/scope\_spec.js_

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

> 可能你已经察觉到了，子作用域也是有自己的 `$watch` 方法的，因为这个 `$watch` 方法是定义在 `Scope.prototype` 上的。这与用户自定义属性的继承机制是一样的：父作用域继承自 `Scope.prototype`，子作用域继承父作用域，`Scope.prototype` 上定义的东西自然就能被每一个作用域访问到了！

上面讨论的所有内容适用于任意深度的作用域层次：

_test/scope\_spec.js_

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

虽然上面定义了很多东西，但实现起来很简单。我们只需要利用 JavaScript 对象继承就可以了，Angular 作用域就是按照 JavaScript 本身的工作方式设计的。实际上，当你创建子作用域时，它的父作用域就会自动成为它的原型了。

> 我们不会花太多时间讨论 JavaScript 中的原型是什么。如果你想复习一下的话，DailyJS 上有很多关于原型和继承的好文章，[prototype](http://dailyjs.com/2012/05/20/js101-prototype/) 和 [inheritance](http://dailyjs.com/2012/05/27/js101-prototype-chains/)。
>
> 译者注：以上两个链接由于已经是 2012 年的链接，已经失效。如有必要，请看 MDN 的说明 [继承与原型链](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

我们在 `Scope`  构造函数中创建 `$new` 函数。这个函数能够为当前作用域创建一个子作用域，并返回这个子作用域：

```js
Scope.prototype.$new = function() {
  var ChildScope = function() {};
  ChildScope.prototype = this;
  var child = new ChildScope();
  return child;
};
```

在这个函数中，我们首先要创建子作用域的构造函数，然后把它赋值到 `ChildScope` 局部变量上。这个构造函数什么都不用做，因此只需要一个空函数就可以了。接着，我们把 `Scope` 设置为 `ChildScope` 的原型。然后使用 `ChildScope` 构造函数创建一个对象，最后返回这个对象就可以。

这个函数虽然没多少代码，但已经能让本节中的所有测试用例都通过了！

> 你也可以使用 ES5 提供的简写函数（shorthand function）`Object.create()` 来构建子作用域。




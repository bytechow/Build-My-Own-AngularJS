### 单向数据绑定（One-Way Data Binding）

属性绑定常常能起到不小用处，而独立作用域绑定最常用于数据绑定：让独立作用域上的作用域属性和其父作用域上解释的表达式进行关联。

数据绑定也允许父子作用域之间进行像普通继承作用域一样的数据共享，但其中有几种重要的区别：

|继承作用域|独立作用域|
|--|--|
|父作用域的所有属性都会传递给子作用域|只有在表达式中提及到的属性才会被共享|
|父子属性是一一对应的关系|子作用域属性在父作用域上可能没有对应属性，但可以使用任何表达式|

Angular 有两种数据绑定的类型：单向数据绑定，可以允许我们往独立作用域传递数值。双向绑定，除了能往独立作用域传递属性值，还可以把属性值的变化反馈给父作用域。

在这两个类型中，应用开发者用得最多的是单向绑定。往下传递数据给指令或组件就是一个典型案例。要用到双向绑定的情况比较少，但我们还是可以在有需要时用到它。

> 现实中大多数应用程序中，双向数据绑定还是会比单向绑定更常见。这是有一个历史原因：单向数据绑定直到 Angular 1.5 版本才被加入，而这时双向绑定已经被广泛应用了。因此，大多数情况下我们很可能会会遇到一个向下的双向数据绑定，而我们实际上可以直接用单向数据绑定代替。

让我们开始探索单向绑定是如何实现的。最简单的单向绑定方式是在作用域定义对象中对属性使用'`<`'符号。这就相当于说：“在父作用域上把这个属性值当作一个 Angular 表达式进行解释”。这个表达式本身不会在作用域定义对象进行定义，但会在指令所对应的 DOM 属性上进行定义。下面就是一个例子，`anAttr`就是通过这种方式进行绑定的，属性值表达式`'42'`，将会被解释为数字`42`：

_test/compile_spec.js_

```js
it('allows binding expression to isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        anAttr: '<'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive an-attr="42"></div>');
    $compile(el)($rootScope);
    
    expect(givenScope.anAttr).toBe(42);
  });
});
```
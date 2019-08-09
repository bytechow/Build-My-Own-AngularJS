### 插入字符串（Interpolating Strings）

做好准备工作之后，我们就可以开始介绍`$interpolate`函数究竟会做些什么事了。

这个函数的基本工作是——接收一个字符串，这个字符串是被`{{ }}`包裹的，可能包含也可能不包含 Angular 表达式。然后会对字符串进行特定处理，最后返回一个函数。这个函数调用之后，会对字符串里面的所有表达式进行重新求值，并生成插值结果。这个函数需要一个上下文对象（通常是一个作用域），表达式将会在这个上下文语境内进行求值：

```js
var interpolateFn = $interpolate('{{a}} and {{b}}');
interpolateFn({a: 1, b: 2}); // => '1 and 2'
```

值得注意的是，这个效果很像`$parse`的作用：`$interpolate`和`parse`服务都会接收一个已经经过了一次“解析”的字符串。它们的返回值也是一个可以在指定作用域语境下进行求值的函数。这种相似性并不出奇，因为`$interpolate`服务实际上只是对`$parse`服务的一层封装：每个被花括号包裹的表达式都会单独使用`$parse`服务进行处理。

在我们讲述这种带有表达式的 interpolation 之前，我们先来看看最简单的一种 interpolation，也就是不带任何表达式的字符串。当传入一个简单的字符串，`$interpolate`应该会返回一个会产出相同字符串的函数：

_test/interpolate_spec.js_

```js
it('produces an identity function for static content', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  
  var interp = $interpolate('hello');
  expect(interp instanceof Function).toBe(true);
  expect(interp()).toEqual('hello');
});
```

要让这个单元测试通过，最简单的方法就是让函数直接返回传入的参数值：

_src/interpolate.js_

```js
this.$get = function() {

  function $interpolate(text) {
    return function interpolationFn() {
      return text;
    };
  }
  
  return $interpolate;
};
````
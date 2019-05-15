### 表达式式绑定（Expression Binding）

第四种也是最后一种往独立作用域绑定的方式是绑定一个表达式，这是通过在作用域定义对象中使用`'&'`符号来实现的。这种方式跟其它两种绑定方式的主要区别是它是主要用来绑定行为而不是绑定数据的：当你使用这个指令，就是提供了一个可供指令在某一个时刻调用的表达式。这对于事件驱动型的指令如`ngClick`来说是尤为有用，同时也有广泛的适用性。

绑定的表达式将会以一个函数的形式出现在一个独立作用域上，而这个函数我们是可以通过调用返回的，例如，一个 link 函数：

_test/compile_spec.js_

```js
it('allows binding an invokable expression on the parent scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myExpr: '&'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.parentFunction = function() {
      return 42;
    };
    var el = $('<div my-directive my-expr="parentFunction() + 1"></div>');
    $compile(el)($rootScope);
    expect(givenScope.myExpr()).toBe(43);
  });
});
```

独立作用域上的`myExpr`函数实际上是`parentFunction() + 1`这个表达式的表达式函数，它会调用父作用域上的`parentFunction`并对返回函数调用结果加1。
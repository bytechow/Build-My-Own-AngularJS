### 在作用域上加入指令控制器（Attaching Directive Controllers on The Scope）

我们已经知道如何往控制器传递作用域对象了，其实我们也可以反过来为作用域添加一个控制器对象。这允许我们进行一个新的应用模式：通过`this`而不是`$scope`来公开控制器数据和函数，同时还可以让它们能够在 DOM 差值表达式使用，也能够在子指令和控制器中使用。

> [Todd Motto的一篇文章](http://toddmotto.com/digging-into-angulars-controller-as-syntax/)对这种应用模式有一个不错的介绍。之后我们再根据文章的内容对这种模式作进一步的解释，这里我们先关注搭建这种模式的基础。

当指令定义对象定义了一个`controllerAs`的 key 时，也就指定了添加到作用域的控制器对象的 key：

_test/compile_spec.js_

```js
it('can be attached on the scope', function() {
  function MyController() {}
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          controller: 'MyController',
          controllerAs: 'myCtrl'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect($rootScope.myCtrl).toBeDefned();
    expect($rootScope.myCtrl instanceof MyController).toBe(true);
  });
});
```

在这个用例中，指令并没有要求获得一个继承的或是独立的作用域，所以获取这个控制器的作用域是`$rootScope`，这更方便我们进行测试。
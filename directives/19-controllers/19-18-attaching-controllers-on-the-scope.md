### 附加控制器到作用域上（Attaching Controllers on The Scope）

本章较早前提到了我们可以通过在指令定义对象中定义`controllerAs`属性来把控制器绑定到作用域上。

实际上，我们还有另一种绑定的方式，这种方式通常会和`ngController`结合起来使用。也就是为控制器构造函数定义一个别称：

```xml
<div ng-controller="TodoController as todoCtrl">
  <!-- ... -->
</div>
```

我们会把这个测试用例放到`ngController`的测试文件中：

_test/directives/ng_controller_spec.js_

```js
it('allows aliasing controller in expression', function() {
  var gotScope;

  function MyController($scope) {
    gotScope = $scope;
  }
  var injector = createInjector(['ng', function($controllerProvider) {
    $controllerProvider.register('MyController', MyController);
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div ng-controller="MyController as myCtrl"></div>');
    $compile(el)($rootScope);
    expect(gotScope.myCtrl).toBeDefned();
    expect(gotScope.myCtrl instanceof MyController).toBe(true);
  });
});
```
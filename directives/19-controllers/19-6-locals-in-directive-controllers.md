### 指令控制器中的本地变量（Locals in Directive Controllers）

虽然我们已经知道了如何实例化一个指令控制器，但指令和控制器之间的联系目前还不存在：控制器虽然在指令中进行实例化，但实际上并没有对指令上的内容进行访问，这让它的价值大大降低了。

我们可以对控制器做一些改变来加强这种联系：

- $scope：指令的作用域对象
- $element: 要应用指令的那个元素本身
- $attrs: 要应用指令的那个元素的属性对象

这些在控制器控制函数中都是可用的：

_test/compile_spec.js_

```js
it('gets scope, element, and attrs through DI', function() {
  var gotScope, gotElement, gotAttrs;

  function MyController($element, $scope, $attrs) {
    gotElement = $element;
    gotScope = $scope;
    gotAttrs = $attrs;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          controller: 'MyController'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive an-attr="abc"></div>');
    $compile(el)($rootScope);
    expect(gotElement[0]).toBe(el[0]);
    expect(gotScope).toBe($rootScope);
    expect(gotAttrs).toBeDefned();
    expect(gotAttrs.anAttr).toEqual('abc');
  });
});
```
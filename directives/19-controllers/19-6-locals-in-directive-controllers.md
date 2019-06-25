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

> 在构造函数的参数中，我们有目的地把`$element`放到`$scope`前面，这是想强调一个事实：这些参数都是通过依赖注入的，它们的排列顺序并不会产生影响。而在链接函数中，这三个参数总是按照`scope`、`element`、`attrs`这样的顺序进行排列，因为链接函数并不使用依赖注入。

要传递这些对象到控制器构造函数里面，我们可以运用早先为`$controller`添加的`locals`支持。我们只需要在控制器循环中创建一个合适的本地对象，并把它传入到`$controller`中。这样，我们让`$scope`、`$element`和`$attrs`变成可以被注入：

_src/compile.js_

```js
_.forEach(controllerDirectives, function(directive) {
  var locals = {
    $scope: scope,
    $element: $element,
    $attrs: attrs
  };
  var controllerName = directive.controller;
  if (controllerName === '@') {
    controllerName = attrs[directive.name];
  }
  $controller(controllerName, locals);
});
```

现在控制器与指令就比以前联系更紧密了。事实上，有了这三个对象，所有我们可以在指令链接函数里做的事情，也都可以在指令控制器中做了。实际上，很多人会选择在指令控制器里中组织指令的代码逻辑，这样对应的链接函数就不用做太多的事了。这样做有一个优点，就是我们不需要实例化指令也可以把控制器变成一个独立的、可以进行单元测试的组件——这是使用链接函数无法做到的。
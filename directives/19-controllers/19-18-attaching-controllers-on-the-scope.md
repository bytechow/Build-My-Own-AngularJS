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

现在，即使我们已经在`ng_controller_spec.js`有了相关的测试，但代码开发却并不会出现在 ngController 里面，而是在`$controller`服务里。在查找控制器的过程中，`$controller`首先会从给予的字符串中提取出真正的控制器名称和别称：

_src/controller.js_

```js
return function(ctrl, locals, later, identifer) {
  // if (_.isString(ctrl)) {
    var match = ctrl.match(/^(\S+)(\s+as\s+(\w+))?/);
    ctrl = match[1];
  //   if (controllers.hasOwnProperty(ctrl)) {
  //     ctrl = controllers[ctrl];
  //   } else if (globals) {
  //     ctrl = window[ctrl];
  //   }
  // }
  // ...
}
```

这个正则表达式会匹配到一组不含空白符的字符作为控制器名称，然后是可能会出现的`as`关键词和它周围的空白符，这个关键词后面会有另一组字符表示控制器的别称：

```js
return function(ctrl, locals, later, identifer) {
  // if (_.isString(ctrl)) {
  //   var match = ctrl.match(/^(\S+)(\s+as\s+(\w+))?/);
  //   ctrl = match[1];
    identifer = identifer || match[3];
  //   if (controllers.hasOwnProperty(ctrl)) {
  //     ctrl = controllers[ctrl];
  //   } else if (globals) {
  //     ctrl = window[ctrl];
  //   }
  // }
  // ...
}
```

> 注意，虽然“controller as”一般都是与 ngController 结合使用，但并不是必须都这样用。这个功能是由`$controller`服务进行提供的，所以你也可以把它用于指令控制器，直接指定类似`MyCtrl as myCtrl`的值作为（指令定义对象的）`controller`属性值，而不用分别指定`controller`和`controllerAs`。
### 独立作用域指令上的控制器（Controllers on Isolate Scope Directives）

乍看之下，独立作用域指令使用的控制器与非独立作用域的并没有太大的不同。然而，与独立作用域有关的一些特性确实会存在一些障碍点，这需要我们特别注意。

在我们开始之前，先介绍一些基础知识：当一个指令拥有一个独立作用域，要注入到控制器中的`$scope`参数的就应该是独立作用域，而不是上层的控制器。

_test/compile_spec.js_

```js
it('gets isolate scope as injected $scope', function() {
  var gotScope;

  function MyController($scope) {
    gotScope = $scope;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          scope: {},
          controller: 'MyController'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(gotScope).not.toBe($rootScope);
  });
});
```

要支持这种行为，我们需要对节点链接函数中的代码进行一点修改。任何的独立作用域都必须在控制器实例化之前创建。我们将仍然会在控制器实例化后继续完成独立作用域创建的剩余工作，这样做的理由我们之后会进行介绍。

一旦我们有了独立作用域对象，我们会把它传递到指令控制器里去。我们应该注意，只有在指令真的需要时才使用独立作用域。所有其他在节点上的“活跃”控制器接收的仍然是非独立的作用域：

_src/compile.js_

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  // var $element = $(linkNode);

  var isolateScope;
  if (newIsolateScopeDirective) {
    isolateScope = scope.$new(true);
    $element.addClass('ng-isolate-scope');
    $element.data('$isolateScope', isolateScope);
  }

  if (controllerDirectives) {
    _.forEach(controllerDirectives, function(directive) {
      var locals = {
        $scope: directive === newIsolateScopeDirective ? isolateScope : scope,
        $element: $element,
        $attrs: attrs
      };
      var controllerName = directive.controller;
      if (controllerName === '@') {
        controllerName = attrs[directive.name];
      }
      $controller(controllerName, locals, directive.controllerAs);
    });
  }

  if (newIsolateScopeDirective) {
    _.forEach(
      newIsolateScopeDirective.$$isolateBindings,
      function(defnition, scopeName) {
        // ...
      }
    )
  }

  // ...

}
```

我们可以吧所有独立作用域的创建代码都移动到控制器实例化代码之前，而我们的这个测试用例也依然可以通过。那我们为什么要像这样将它们分隔开来？

原因在于一个与独立作用域相关的特性，叫做`bindToController`，这个特性也是我们下一步要开发的。`bindToController`是一个可以在指令定义对象中设置的标识，可以控制在哪里加入所有的独立作用域绑定。在上一章，我们看到过的所有使用`@、<、=`或者`&`前缀的绑定都会加入到独立作用域上。然而，如果指令上设置了`bindToController`标识，那这些绑定将会加入到控制器对象而不是独立作用域上。当这个标识与`controllerAs`选项结合使用时尤其有用，这可以让控制器对象和独立作用域绑定均能被子元素使用。

现在当我们需要设置这些特性时，我们就会遇到一个“先有鸡还是先有蛋”的问题：


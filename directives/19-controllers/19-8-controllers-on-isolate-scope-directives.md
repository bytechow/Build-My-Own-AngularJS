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

我们可以吧所有独立作用域的创建代码都移动到控制器实例化代码之前，而我们的这个测试用例也依然可以通过。那我们为什么要将它们分隔开来？

原因在于一个与独立作用域相关的特性，叫做`bindToController`，这个特性也是我们下一步要开发的。`bindToController`是一个可以在指令定义对象中设置的标识，可以控制在哪里加入所有的独立作用域绑定。在上一章，我们看到过的所有使用`@、<、=`或者`&`前缀的绑定都会加入到独立作用域上。然而，如果指令上设置了`bindToController`标识，那这些绑定将会加入到控制器对象而不是独立作用域上。当这个标识与`controllerAs`选项结合使用时尤其有用，这可以让控制器对象和独立作用域绑定均能被子元素使用。

现在当我们需要设置这些特性时，我们就会遇到一个“先有鸡还是先有蛋”的问题：

1. 独立作用域绑定应该在控制器构造函数被调用之前就存在，因为构造函数运行时可能会需要这些绑定已经准备好了。

2. 如果使用了`bindToController`，独立作用域绑定一定要绑定到控制器对象上，也就是说我们在设置独立作用域绑定之前必须已经有控制器对象了。

这意味着我们需要在实际调用控制器构造前之前拥有一个控制器对象。得益于 Javascript 的灵活性，我们是有办法实现的，但要实现还需要先做一些工作。

我们先加入两个测试用例来展示我们是怎么解决的。当调用了一个控制器构造函数，所有独立作用域绑定必须已经存在于作用域上：

_test/compile_spec.js_

```js
it('has isolate scope bindings available during construction', function() {
  var gotMyAttr;
  function MyController($scope) {
    gotMyAttr = $scope.myAttr;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          scope: {
            myAttr: '@myDirective'
          },
          controller: 'MyController'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive="abc"></div>');
    $compile(el)($rootScope);
    expect(gotMyAttr).toEqual('abc');
  });
});
```

另一方面，如果启用了`bindToController`，独立作用域绑定将会附加到控制器实例上，而不是`$scope`上。当调用控制器构造函数时，属性已经存在于`this`上：

_test/compile_spec.js_

```js
it('can bind isolate scope bindings directly to self', function() {
  var gotMyAttr;
  function MyController() {
    gotMyAttr = this.myAttr;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          scope: {
            myAttr: '@myDirective'
          },
          controller: 'MyController',
          bindToController: true
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive="abc"></div>');
    $compile(el)($rootScope);
    expect(gotMyAttr).toEqual('abc');
  });
});
```

最初这两个测试用例都无法通过，直到我们完成功能为止。所以，现在我们继续深入探讨细节。

`$controller`函数接收的第三个可选参数叫做`later`，它可以让函数返回一个“非完全构造”的控制器而不是一个完整构造好的控制器。

"非完全构造"的含义是控制器对象已经存在，但这个控制器构造函数尚未调用。具体而言，这种情况下`$controller`的返回值拥有下面的特性：

- 它是一个函数，调用时会调用控制器构造函数
- 它有一个叫`instance`的属性并指向控制器对象。

这种可以延迟调用的构造函数给了`$controller`一个机会在创建控制器对象和调用构造函数之间的时间做一些工作——也就是设置独立作用域绑定。

_test/controller_spec.js_

```js
it('can return a semi-constructed controller', function() {
  var injector = createInjector(['ng']);
  var $controller = injector.get('$controller');

  function MyController() {
    this.constructed = true;
    this.myAttrWhenConstructed = this.myAttr;
  }

  var controller = $controller(MyController, null, true);
  
  expect(controller.constructed).toBeUndefned();
  expect(controller.instance).toBeDefned();
  
  controller.instance.myAttr = 42;
  var actualController = controller();
  
  expect(actualController.constructed).toBeDefned();
  expect(actualController.myAttrWhenConstructed).toBe(42);
});
```

由于我们在`$controller`中引入`later`参数，`identifier`参数需要推后到第四个位置：

_src/controller.js_

```js
this.$get = ['$injector', function($injector) {

  return function(ctrl, locals, later, identifier) {
    // ...
  };
  
}];
```

我们希望让目前的测试用例通过，所以在`compile.js`文件中暂时先把`later`的值默认设置为`false`。稍后我们会改过来：

_src/compile.js_

```js
_.forEach(controllerDirectives, function(directive) {
  // var locals = {
  //   $scope: directive === newIsolateScopeDirective ? isolateScope : scope,
  //   $element: $element,
  //   $attrs: attrs
  // };
  // var controllerName = directive.controller;
  // if (controllerName === '@') {
  //   controllerName = attrs[directive.name];
  // }
  $controller(controllerName, locals, false, directive.controllerAs);
});
```

> `later`和`identifier`都被设计成只允许内部调用的，而不能被应用开发者直接使用。如果你确实需要使用它们，你得意识到它们可能会在未来改变设置消失，因为它们仅被考虑用在实现细节上。

在`$controller`，我们现在根据这个标识可以决定是做一般的实例化还是其他：

_src/controller.js_

```js
return function(ctrl, locals, later, identifer) {
  // if (_.isString(ctrl)) {
  //   if (controllers.hasOwnProperty(ctrl)) {
  //     ctrl = controllers[ctrl];
  //   } else if (globals) {
  //     ctrl = window[ctrl];
  //   }
  // }
  var instance;
  if (later) {
    
  } else {
    instance = $injector.instantiate(ctrl, locals);
    // if (identifer) {
    //   addToScope(locals, identifer, instance);
    // }
    // return instance;
  }
};
```
### 指令控制器（Directive Controllers）

目前，我们已经拥有一个可以注册、查找和实例化控制的`$controller`服务了。接下来，我们可以开始关注使用控制器的情况了。这也是指令发挥作用的地方。

你可以为指令指定一个控制器，只需要在指令定义对象中指定一个`controller`属性并且属性值是一个控制器构造函数即可。这个控制器构造函数会在指令链接时进行实例化。

我们先为此准备一个测试，同时也要再在`compile_spec.js`文件中加入`describe`代码块，并放到在本章中所有与控制器相关的测试用例中：

_test/compile_spec.js_

```js
describe('controllers', function() {

  it('can be attached to directives as functions', function() {
    var controllerInvoked;
    var injector = makeInjectorWithDirectives('myDirective', function() {
        return {
          controller: function MyController() {
            controllerInvoked = true;
          }
        };
      });
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div my-directive></div>');
      $compile(el)($rootScope);
      expect(controllerInvoked).toBe(true);
    });
  });
  
});
```

`controller`属性也可以是一个字符串，这个字符串指向一个之前注册过的控制器构造函数：

_test/compile_spec.js_

```js
it('can be attached to directives as string references', function() {
  var controllerInvoked;
  function MyController() {
    controllerInvoked = true;
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
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(controllerInvoked).toBe(true);
  });
});
```

指令控制器都是独立地为各个指令进行实例化的，但并不限制同一个元素上有多个不同的指令控制器。下面我们测试应用了一个元素包含两个指令，而这两个指令都有自己的控制器：

_test/compile_spec.js_

```js
it('can be applied in the same element independent of each other', function() {
  var controllerInvoked;
  var otherControllerInvoked;
  function MyController() {
    controllerInvoked = true;
  }
  function MyOtherController() {
    otherControllerInvoked = true;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $controllerProvider.register('MyOtherController', MyOtherController);
      $compileProvider.directive('myDirective', function() {
        return {
          controller: 'MyController'
        };
      });
      $compileProvider.directive('myOtherDirective', function() {
        return {
          controller: 'MyOtherController'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    $compile(el)($rootScope);
    expect(controllerInvoked).toBe(true);
    expect(otherControllerInvoked).toBe(true);
  });
});
```

Angular 也不限制多次使用同一个控制器构造函数。即使在同一个元素上两个指令使用了同一个控制器构造函数，每个指令本身还是会获取控制器的一个属于自己的实例：

_test/compile_spec.js_

```js
it('can be applied to different directives, as different instances', function() {
  var invocations = 0;
  function MyController() {
    invocations++;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          controller: 'MyController'
        };
      });
      $compileProvider.directive('myOtherDirective', function() {
        return {
          controller: 'MyController'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    $compile(el)($rootScope);
    expect(invocations).toBe(2);
  });
});
```

下面我们需要这些单元测试通过。首先，当我们在`applyDirectivesToNode`的编译阶段对指令进行遍历时，我们需要对有控制器的指令进行收集：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  var $compileNode = $(compileNode);
  var terminalPriority = -Number.MAX_VALUE;
  var terminal = false;
  var preLinkFns = [],
    postLinkFns = [];
  var newScopeDirective, newIsolateScopeDirective;
  var controllerDirectives;

  function addLinkFns(preLinkFn, postLinkFn, attrStart, attrEnd, isolateScope) {
    // ...
  }

  _.forEach(directives, function(directive) {
  
    // ...
  
    if (directive.controller) {
      controllerDirectives = controllerDirectives || {};
      controllerDirectives[directive.name] = directive;
    }
  });
  
  // ...

}
```

这里我们构建了一个`controllerDirectives`对象，来存放带有控制器的指令，key 是指令名称，而 value 就是对应的指令对象。

有了这个对象的帮助，在链接阶段，当一个节点进行链接时，我们就知道需要进行实例化的控制器有哪些。我们可以使用`$controller`服务来完成这个功能。它应该能够处理关于指令控制器的任何值，无论是一个构造函数还是一个代表构造函数的名称字符串：

_src/compile.js_

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  // var $element = $(linkNode);
  
  if (controllerDirectives) {
    _.forEach(controllerDirectives, function(directive) {
      $controller(directive.controller);
    });
  }

  // ...
  
}
```

在代码起作用之前，我们需要把`$controller`服务注入到`$compileProvider.$get`中：

```js
this.$get = ['$injector', '$parse', '$controller', '$rootScope',
function($injector, $parse, $controller, $rootScope) {
```

这里我们所有的测试用例就都通过了。指令控制器会使用`$controller`服务来实例化指令控制器，而这只需要把`$compile`和`$controller`结合在一起就可以了。

关于指令控制器的整合还有一个有趣的特性，就是当你有一个属性式指令并使用`‘@’`指定控制器名称，那么就会使用 DOM 中的指令属性值来查找控制器。这在指令被使用而不是被注册时指定指令控制器的情况下十分有用。实际上，这种特性允许我们对同一个指令绑定不同的控制器。

_test/compile_spec.js_

```js
it('can be aliased with @ when given in directive attribute', function() {
  var controllerInvoked;
  function MyController() {
    controllerInvoked = true;
  }
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          controller: '@'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive="MyController"></div>');
    $compile(el)($rootScope);
    expect(controllerInvoked).toBe(true);
  });
});
```

这个支持会放到节点链接函数，当使用`'@'`进行指定时，控制器名称就会被替代为 DOM 属性上的值：

_src/compile.js_

```js
_.forEach(controllerDirectives, function(directive) {
  var controllerName = directive.controller;
  if (controllerName === '@') {
  controllerName = attrs[directive.name];
  }
  $controller(controllerName);
});
```
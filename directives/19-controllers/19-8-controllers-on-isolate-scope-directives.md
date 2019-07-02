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

我们要做的有两个步骤：

1. 创建一个原型基于构造函数的新对象。使用`Object.create`就很合适。
2. 返回一个“非完全构造”的控制器：是一个能真正能在后面被调用的函数，这个函数有一个`instance`属性存放着对象实例。

当我们最终要调用构造函数时，我们不应该使用`$injector.instantiate`，因为我们在那个时间节点实际上还没有实例化任何东西。我们可以把它们当做是普通的依赖注入函数，使用`$injector.invoke`进行调用。我们只需要把构造函数作为`self`参数传入，那`this`绑定就没有问题了。

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
  // var instance;
  // if (later) {
    instance = Object.create(ctrl);
    return _.extend(function() {
      $injector.invoke(ctrl, instance, locals);
      return instance;
    }, {
      instance: instance
    });
  // } else {
  //   instance = $injector.instantiate(ctrl, locals);
  //   if (identifer) {
  //     addToScope(locals, identifer, instance);
  //   }
  //   return instance;
  // }
};
```

因为`$controller`支持依赖注入，那么就可以使用数组形态的依赖注入来注入第一个参数，而不是一个函数式注入。他应该依然支持`later`标识：

_test/controller_spec.js_

```js
it('can return a semi-constructed ctrl when using array injection', function() {
  var injector = createInjector(['ng', function($provide) {
    $provide.constant('aDep', 42);
  }]);
  var $controller = injector.get('$controller');

  function MyController(aDep) {
    this.aDep = aDep;
    this.constructed = true;
  }
  
  var controller = $controller(['aDep', MyController], null, true);
  expect(controller.constructed).toBeUndefned();
  var actualController = controller();
  expect(actualController.constructed).toBeDefned();
  expect(actualController.aDep).toBe(42);
});
```

因为我们需要传递控制器原型给`Object.create`，如果注入的是一个数组，我们需要找到注入的控制器：

_src/controller.js_

```js
var ctrlConstructor = _.isArray(ctrl) ? _.last(ctrl) : ctrl;
instance = Object.create(ctrlConstructor.prototype);
// return _.extend(function() {
//   $injector.invoke(ctrl, instance, locals);
//   return instance;
// }, {
//   instance: instance
// });
```

结合第三个参数`later`和第四个参数`identifier`，如果我们要求，可以让`$controller`在作用域上绑定一个“未完成的”控制器对象：

_test/controller_spec.js_

```js
it('can bind semi-constructed controller to scope', function() {
  var injector = createInjector(['ng']);
  var $controller = injector.get('$controller');

  function MyController() {
  }
  var scope = {};

  var controller = $controller(MyController, {
    $scope: scope
  }, true, 'myCtrl');
  expect(scope.myCtrl).toBe(controller.instance);
});
```

> 这里我们没有用一个真实的作用域对象而只是一个普通对象。因为对于测试目标来说，这两者没有区别。

我们通过调用之前在“渴望的构造函数”（）中已经调用过的`addToScope`帮助函数来完成这一任务：

_src/controller.js_

```js
if (later) {
  // var ctrlConstructor = _.isArray(ctrl) ? _.last(ctrl) : ctrl;
  // instance = Object.create(ctrlConstructor.prototype);
  if (identifer) {
    addToScope(locals, identifer, instance);
  }
  // return _.extend(function() {
  //   $injector.invoke(ctrl, instance, locals);
  //   return instance;
  // }, {
  //   instance: instance
  // });
} else {
  // instance = $injector.instantiate(ctrl, locals);
  // if (identifer) {
  //   addToScope(locals, identifer, instance);
  // }
  // return instance;
}
```

现在我们已经有了`$controller`需要有的基础，我们可以在`$compile`中通过做一些调整来把东西都整合在一起。

首先，我们需要引入一个变量来存放“非完全构造”的控制器函数。这个会在`applyDirectivesToNode`函数的顶层进行定义：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  var $compileNode = $(compileNode);
  var preLinkFns = [],
    postLinkFns = [],
    controllers = {};
  // ...
}
```

当我们构建了控制器，我们会把“非完全构造”的函数放到这个对象中去。还要注意到我们现在对`$controller`函数的第三个参数传递了`true`以触发`later`标识：

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
  controllers[directive.name] =
    $controller(controllerName, locals, true, directive.controllerAs);
});
````

然后，在我们设置好独立作用域绑定之后、在调用预链接函数之前，我们会调用“非完全构造”的控制器函数，它会调用真正的控制器构造函数：

```js
// if (newIsolateScopeDirective) {
//   _.forEach(
//     newIsolateScopeDirective.$$isolateBindings,
//     function(defnition, scopeName) {
//       // ...
//     }
//   );
// }

_.forEach(controllers, function(controller) {
  controller();
});

// _.forEach(preLinkFns, function(linkFn) {
//   linkFn(linkFn.isolateScope ? isolateScope : scope, $element, attrs);
// });
```

实际上，我们现在是在控制器是“非完全构造”状态下设置好独立作用域绑定。

这样我们两个“高级”测试用例中的第一个就通过了，但另一个为了`bindToController`的测试用例依旧还没通过。我们要做的是对独立作用域绑定的初始化进行扩展，让它可以处理两种不同的情况：普通的独立作用域绑定和作用域绑定最终会绑定到控制器上（当`bindToController`为`true`）。

要让这做起来简单一点，我们需要对作用域绑定的解析代码进行一点改变。我们现在的做法是当我们初始化时对指令设置一个`$$isolateBindings`属性。我们应该将这个属性统一为一个`$$bindings`的属性，这个属性我们可以控制作用域绑定和控制器绑定。要实例化这个属性，我们会使用一个新的帮助函数叫`parseDirectiveBindings`:

_src/controller.js_

```js
return _.map(factories, function(factory, i) {
  // var directive = $injector.invoke(factory);
  // directive.restrict = directive.restrict || 'EA';
  // directive.priority = directive.priority || 0;
  // if (directive.link && !directive.compile) {
  //   directive.compile = _.constant(directive.link);
  // }
  directive.$$bindings = parseDirectiveBindings(directive);
  // directive.name = directive.name || name;
  // directive.index = i;
  // return directive;
});
```

第一个版本的`parseDirectiveBindings`仅可以调用已经存在的`parseIsolateBindings`函数，并把返回值放到新绑定对象的`isolateScope`属性：

_src/compile.js_

```js
function parseDirectiveBindings(directive) {
  var bindings = {};
  if (_.isObject(directive.scope)) {
    bindings.isolateScope = parseIsolateBindings(directive.scope);
  }
  return bindings;
}
```

现在将目光转到链接期间的绑定初始化，这里我们也需要做一些重构工作。让我们把绑定初始化代码从`nodeLinkFn`中抽离到一个独立的函数中。我们可以将这个函数命名为`initializeDirectiveBindings`。它包含了之前我们写的用于初始化的循环代码，并接收一个它所需的参数来完成它的工作：

```js
function initializeDirectiveBindings(scope, attrs, bindings, isolateScope) {
  _.forEach(bindings, function(defnition, scopeName) {
    var attrName = defnition.attrName;
    var parentGet, unwatch;
    switch (defnition.mode) {
      case '@':
        attrs.$observe(attrName, function(newAttrValue) {
          isolateScope[scopeName] = newAttrValue;
        });
        if (attrs[attrName]) {
          isolateScope[scopeName] = attrs[attrName];
        }
        break;
      case '<':
        if (defnition.optional && !attrs[attrName]) {
          break;
        }
        parentGet = $parse(attrs[attrName]);
        isolateScope[scopeName] = parentGet(scope);
        unwatch = scope.$watch(parentGet, function(newValue) {
          isolateScope[scopeName] = newValue;
        });
        isolateScope.$on('$destroy', unwatch);
        break;
      case '=':
        if (defnition.optional && !attrs[attrName]) {
          break;
        }
        parentGet = $parse(attrs[attrName]);
        var lastValue = isolateScope[scopeName] = parentGet(scope)
        var parentValueWatch = function() {
          var parentValue = parentGet(scope);
          if (isolateScope[scopeName] !== parentValue) {
            if (parentValue !== lastValue) {
              isolateScope[scopeName] = parentValue;
            } else {
              parentValue = isolateScope[scopeName];
              parentGet.assign(scope, parentValue);
            }
          }
          lastValue = parentValue;
          return lastValue;
        };
        if (defnition.collection) {
          unwatch = scope.$watchCollection(attrs[attrName], parentValueWatch);
        } else {
          unwatch = scope.$watch(parentValueWatch);
        }
        isolateScope.$on('$destroy', unwatch);
        break;
      case '&':
        var parentExpr = $parse(attrs[attrName]);
        if (parentExpr === _.noop && defnition.optional) {
          break;
        }
        isolateScope[scopeName] = function(locals) {
          return parentExpr(scope, locals);
        };
        break;
    }
  });
}
```

而`nodeLinkFn`本身，现在只需要做的就是调用这个新函数而无需再加入之前那段长长的循环代码了。对于绑定（bindings）来说，我们会传递在新的`parseDirectiveBindings`函数中创建的独立作用域绑定：

```js
if (newIsolateScopeDirective) {
  initializeDirectiveBindings(
    scope,
    attrs,
    newIsolateScopeDirective.$$bindings.isolateScope,
    isolateScope
  );
}
```

现在我们准备对这个进行扩展，以让其也兼容`bindToController`标识。在`parseDirectiveBindings`中，如果这个标识是`true`，我们会将绑定（bindings）存放到`bindToController`而不是`isolateScope`属性：

```js
function parseDirectiveBindings(directive) {
  // var bindings = {};
  // if (_.isObject(directive.scope)) {
    if (directive.bindToController) {
      bindings.bindToController = parseIsolateBindings(directive.scope);
    } else {
      // bindings.isolateScope = parseIsolateBindings(directive.scope);
    }
  // }
  // return bindings;
}
```

这些新的绑定会在节点链接函数中控制器被实例化之前初始化好。我们只有在有独立作用域指令并且有控制器的情况下才会这样做。我们可以重用之前抽离出来的`initializeDirectiveBindings`函数：

```js
if (newIsolateScopeDirective && controllers[newIsolateScopeDirective.name]) {
  initializeDirectiveBindings(
    scope,
    attrs,
    newIsolateScopeDirective.$$bindings.bindToController,
    isolateScope
  );
}

// _.forEach(controllers, function(controller) {
//   controller();
// });
```

剩下来的事情就是这些绑定现在依然附加在独立作用域对象中，但关键是它们应该被附加到控制器上！`initializeDirectiveBindings`函数应该接收多一个参数，我们把它称为`destination`，这个参数会指向一个对象，而所有的数据应该绑定到这个对象上。这个对象会在不同的绑定类型中被用于赋值对象。我们依然会传入 isolateScope，但我们会称它为`newScope`，它只会被用于注册`$destroy`事件，在需要取消注册观察者时：

```js
function initializeDirectiveBindings(
  scope, attrs, destination, bindings, newScope) {
  _.forEach(bindings, function(defnition, scopeName) {
  //   var attrName = defnition.attrName;
    switch (defnition.mode) {
      case '@':
  //       attrs.$observe(attrName, function(newAttrValue) {
          destination[scopeName] = newAttrValue;
        // });
        // if (attrs[attrName]) {
          destination[scopeName] = attrs[attrName];
      //   }
      //   break;
      // case '<':
      //   if (defnition.optional && !attrs[attrName]) {
      //     break;
      //   }
      //   parentGet = $parse(attrs[attrName]);
        destination[scopeName] = parentGet(scope);
      //   unwatch = scope.$watch(parentGet, function(newValue) {
          destination[scopeName] = newValue;
        // });
        newScope.$on('$destroy', unwatch);
        break;
      case '=':
        // if (defnition.optional && !attrs[attrName]) {
        //   break;
        // }
        // var parentGet = $parse(attrs[attrName]);
        var lastValue = destination[scopeName] = parentGet(scope);
        var parentValueWatch = function() {
          var parentValue = parentGet(scope);
          if (destination[scopeName] !== parentValue) {
            if (parentValue !== lastValue) {
              destination[scopeName] = parentValue;
            } else {
              parentValue = destination[scopeName];
              // parentGet.assign(scope, parentValue);
            }
          }
          lastValue = parentValue;
          return lastValue;
        };
        // var unwatch;
        // if (defnition.collection) {
        //   unwatch = scope.$watchCollection(attrs[attrName], parentValueWatch);
        // } else {
        //   unwatch = scope.$watch(parentValueWatch);
        // }
        newScope.$on('$destroy', unwatch);
        break;
      case '&':
        // var parentExpr = $parse(attrs[attrName]);
        // if (parentExpr === _.noop && defnition.optional) {
        //   break;
        // }
        destination[scopeName] = function(locals) {
          return parentExpr(scope, locals);
        };
        break;
    }
  });
}
```
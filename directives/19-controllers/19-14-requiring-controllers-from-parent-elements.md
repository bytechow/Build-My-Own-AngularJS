### 注入父元素上的指令（Requiring Controllers from Parent Elements）

从兄弟指令上引入控制器是一种情况，但它实际上是非常局限的：我们现在跟元素树的其他指令进行协作（在作用域对象上共享数据以外的）。

`require`标识实际上我们之前看到的更灵活，它允许我们注入父元素上的指令控制器而不仅仅是当前元素上的。这种注入模式可以通过在要注入的指令名称前面加`^`来实现：

_test/compile_spec.js_

```js
it('can be required from a parent directive', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: '^myDirective',
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-other-directive></div></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```

这里的`myOtherDirective`请求注入`^myDirective`，后者是从父元素上找到的指令。

当使用了`^`前缀，就不仅会从父元素查找指令，还会在当前元素查找指令（实际上第一个要检索的就是当前元素）。`^`的准确含义是“当前元素或当前元素的父元素”。

_test/compile_spec.js_

```js
it('fnds from sibling directive when requiring with parent prefx', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: '^myDirective',
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```

要让这样的`require`系统起效，我们不能只依靠在`applyDirectivesToNode`上的`controllers`对象，因为它只能访问到当前元素上的控制器。我们的控制器查找代码现在也需要处理 DOM 结构了。要达成这个目标，第一步就是要传递当前元素给`getControllers`函数：

_src/compile.js_

```js
_.forEach(preLinkFns, function(linkFn) {
  linkFn(
    // linkFn.isolateScope ? isolateScope : scope,
    // $element,
    // attrs,
    linkFn.require && getControllers(linkFn.require, $element)
  );
});
// if (childLinkFn) {
//   childLinkFn(scope, linkNode.childNodes);
// }
// _.forEachRight(postLinkFns, function(linkFn) {
//   linkFn(
//     linkFn.isolateScope ? isolateScope : scope,
//     $element,
//     attrs,
//     linkFn.require && getControllers(linkFn.require, $element)
//   );
// });
```

在`getControllers`的递归调用中，我们也需要传递这个参数：

```js
function getControllers(require, $element) {
  // if (_.isArray(require)) {
    return _.map(require, function(r) {
      return getControllers(r, $element);
    });
  // } else if (_.isObject(require)) {
    return _.mapValues(require, function(r) {
      return getControllers(r, $element);
    });
  // } else {
    // ...
  // }
}
```

在`getControllers`可以找到它所需要的之前，我们需要在创建控制器时对 DOM 加入一下信息。我们创建的任何一个控制器时，应该对相应的 DOM 节点加入 jQuery 数据：

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
  var controller =
    // $controller(controllerName, locals, true, directive.controllerAs);
  controllers[directive.name] = controller;
  $element.data('$' + directive.name + 'Controller', controller.instance);
});
```

现在，在`getControllers`中应该尝试去匹配`require`参数值，看是否以`^`为前缀。如果没有这样的前缀，我们跟之前一样处理就可以了，但如果有，我们需要做一些不同的处理：

```js
function getControllers(require, $element) {
  // if (_.isArray(require)) {
  //   return _.map(require, function(r) {
  //     return getControllers(r, $element);
  //   });
  // } else if (_.isObject(require)) {
  //   return _.mapValues(require, function(r) {
  //     return getControllers(r, $element);
  //   });
  // } else {
  //   var value;
    var match = require.match(/^(\^)?/);
    require = require.substring(match[0].length);
    if (match[1]) {
      
    } else {
      if (controllers[require]) {
        value = controllers[require].instance;
      }
    }
  //   if (!value) {
  //     throw 'Controller ' + require + ' required by directive, cannot be found!';
  //   }
  //   return value;
  // }
}
```

如果有`^`前缀，我们要做的就是通过 DOM 查找 jQuery data 是匹配的指令名字的元素（或者查找到 DOM 树的顶部）：

```js
function getControllers(require, $element) {
  // if (_.isArray(require)) {
  //   return _.map(require, function(r) {
  //     return getControllers(r, $element);
  //   });
  // } else if (_.isObject(require)) {
  //   return _.mapValues(require, function(r) {
  //     return getControllers(r, $element);
  //   });
  // } else {
  //   var value;
  //   var match = require.match(/^(\^)?/);
  //   require = require.substring(match[0].length);
  //   if (match[1]) {
      while ($element.length) {
        value = $element.data('$' + require + 'Controller');
        if (value) {
          break;
        } else {
          $element = $element.parent();
        }
      }
  //   } else {
  //     if (controllers[require]) {
  //       value = controllers[require].instance;
  //     }
  //   }
  //   if (!value) {
  //     throw 'Controller ' + require + ' required by directive, cannot be found!';
  //   }
  //   return value;
  // }
}
```

除了`^`，`require`属性还支持`^^`前缀。它跟`^`的相同之处在于都可以从父元素找到对应的指令控制器：

_test/compile_spec.js_

```js
it('can be required from a parent directive with ^^', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: '^^myDirective',
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-other-directive></div></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```

不同之处在于，`^^`并不会在指令的兄弟元素中进行寻找控制器，而直接是从它的父元素开始查起：

```js
it('does not find from sibling directive when requiring with ^^', function() {
  function MyController() {}
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: '^^myDirective',
        link: function(scope, element, attrs, myController) {}
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    expect(function() {
      $compile(el)($rootScope);
    }).toThrow();
  });
});
```

在上面的测试用例中，我们希望在链接阶段能够抛出异常，因为我们请求注入一个兄弟指令（应用于同一个元素的指令），但我们使用了`^^`前缀所以我们无法找到它。

`getControllers`上的正则表达式需要对可能出现的第二个`^`符号进行适配，如果发现有两个`^`符号，则需要从当前元素的父元素开始查找指令，而不是当前元素：

_src/compile.js_

```js
function getControllers(require, $element) {
  // if (_.isArray(require)) {
  //   return _.map(require, function(r) {
  //     return getControllers(r, $element);
  //   });
  // } else if (_.isObject(require)) {
  //   return _.mapValues(require, function(r) {
  //     return getControllers(r, $element);
  //   });
  // } else {
  //   var value;
  //   var match = require.match(/^(\^\^?)?/);
  //   require = require.substring(match[0].length);
  //   if (match[1]) {
      if (match[1] === '^^') {
        $element = $element.parent();
      }
  //     while ($element.length) {
  //       value = $element.data('$' + require + 'Controller');
  //       if (value) {
  //         break;
  //       } else {
  //         $element = $element.parent();
  //       }
  //     }
  //   } else {
  //     if (controllers[require]) {
  //       value = controllers[require].instance;
  //     }
  //   }
  //   if (!value) {
  //     throw 'Controller ' + require + ' required by directive, cannot be found!';
  //   }
  //   return value;
  // }
}
```

当使用对象语法来注入控制器是，我们需要把前缀直接作为属性值，形成下面这样的简写形式：

```js
require: {
  myDirective: '^'
}
```

下面是一个相关的测试用例：

_test/compile_spec.js_

```js
it('can be required from parent in object form', function() {
  function MyController() {}
  var gotControllers;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: {
          myDirective: '^'
        },
        link: function(scope, element, attrs, controllers) {
          gotControllers = controllers;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-other-directive></div></div>');
    $compile(el)($rootScope);
    expect(gotControllers.myDirective instanceof MyController).toBe(true);
  });
});
```

我们可以在`getDirectiveRequire`函数的指令实例化期间，把这种注册方式进行标准化。我们需要使用与`getControllers`函数中的同一个正则表达式，所以我们把它拿到外面来定义：

```js
var REQUIRE_PREFIX_REGEXP = /^(\^\^?)?/;
```

之后我们在`getControllers`函数的开始部分就使用这个变量：

```js
var match = require.match(REQUIRE_PREFIX_REGEXP);
```

在`getDirectiveRequire`中，我们使用这个正则表达式变量来提取对象属性值中的前缀。然后我们会看提取前缀之后还剩些什么。如果没剩下什么，则用 key 来填充。因为`myDirective: '^'`会变成`myDirective: '^myDirective'`，等等。

_src/compile.js_

```js
function getDirectiveRequire(directive, name) {
  // var require = directive.require || (directive.controller && name);
  // if (!_.isArray(require) && _.isObject(require)) {
  //   _.forEach(require, function(value, key) {
      var prefx = value.match(REQUIRE_PREFIX_REGEXP);
      var name = value.substring(prefx[0].length);
      if (!name) {
        require[key] = prefx[0] + key;
      }
  //   });
  // }
  // return require;
}
```
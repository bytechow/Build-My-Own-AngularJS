### 可选的控制器注入（Optionally Requiring Controllers）

我们目前的`require`代码遇到找不到所需控制器的情况都会抛出异常。实际上，我们也可以选择问号作为前缀，这样我们就不需要一定要找到对应的控制器了。如果你这样做了，即使找不到控制器，我们也不会抛出异常，而是会把`null`作为控制器的值：

_test/compile_spec.js_

```js
it('does not throw on required missing controller when optional', function() {
  var gotCtrl;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        require: '?noSuchDirective',
        link: function(scope, element, attrs, ctrl) {
          gotCtrl = ctrl;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(gotCtrl).toBe(null);
  });
});
```

`REQUIRE_PREFIX_REGEXP`表达式需要允许接收一个问号：

```js
var REQUIRE_PREFIX_REGEXP = /^(\^\^?)?(\?)?/;
```

在`getControllers`中，我们只在没有给出问号前缀时，对无法找到控制器的情况抛出异常。当带上了问号前缀时，我们应该在控制器无法找到时返回一个`null`而不是`undefined`：

_src/controller.js_

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
  //   var match = require.match(REQUIRE_PREFIX_REGEXP);
    var optional = match[2];
    // require = require.substring(match[0].length);
    // if (match[1]) {
    //   if (match[1] === '^^') {
    //     $element = $element.parent();
    //   }
    //   while ($element.length) {
    //     value = $element.data('$' + require + 'Controller');
    //     if (value) {
    //       break;
    //     } else {
    //       $element = $element.parent();
    //     }
    //   }
    // } else {
    //   if (controllers[require]) {
    //     value = controllers[require].instance;
    //   }
    // }
    if (!value && !optional) {
    //   throw 'Controller ' + require + ' required by directive, cannot be found!';
    }
    return value || null;
  // }
}
```

`require`还剩一点需要补充，是跟`^`、`^^`和`?`前缀有关的。实际在 Angular 中把`?`放到`^`（`^^`）的前面或后面都是允许的。也就是说，`?^`和`^?`都是有效的：

_test/compile_spec.js_

```js
it('allows optional marker after parent marker', function() {
  var gotCtrl;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        require: '^?noSuchDirective',
        link: function(scope, element, attrs, ctrl) {
          gotCtrl = ctrl;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(gotCtrl).toBe(null);
  });
});

it('allows optional marker before parent marker', function() {
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
        require: '?^myDirective',
        link: function(scope, element, attrs, ctrl) {
          gotMyController = ctrl;
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

这样的话，我们的正则表达式也得适配这两种形式。

```js
var REQUIRE_PREFIX_REGEXP = /^(\^\^?)?(\?)?(\^\^?)?/;
```

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
  //   var match = require.match(REQUIRE_PREFIX_REGEXP);
  //   var optional = match[2];
  //   require = require.substring(match[0].length);
    if (match[1] || match[3]) {
      if (match[3] && !match[1]) {
        match[1] = match[3];
      }
  //     if (match[1] === '^^') {
  //       $element = $element.parent();
  //     }
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
  //   if (!value && !optional) {
  //     throw 'Controller ' + require + ' required by directive, cannot be found!';
  //   }
  //   return value || null;
  // }
}
```
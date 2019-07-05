### 使用对象形式来引入多个控制器（Requiring Multiple Controllers as an Object）

当你需要引入几个其他指令的控制器，如果要使用数组的索引对指定的控制器进行访问并不是十分方便。为了解决这个问题，我们提供了一种用于引入多个控制器的替代方式，就是使用对象进行引入。也就是说，你在链接函数中获取到的是一个对象，对象中的 key 就是要引入的指令名称，而 value 是指令的控制器。

_test/compile_spec.js_

```js
it('can be required as an object', function() {
  function MyController() {}

  function MyOtherController() {}
  var gotControllers;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: true,
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        scope: true,
        controller: MyOtherController
      };
    });
    $compileProvider.directive('myThirdDirective', function() {
      return {
        require: {
          myDirective: 'myDirective',
          myOtherDirective: 'myOtherDirective'
        },
        link: function(scope, element, attrs, controllers) {
          gotControllers = controllers;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive my-third-directive></div>');
    $compile(el)($rootScope);
    expect(gotControllers).toBeDefned();
    expect(gotControllers.myDirective instanceof MyController).toBe(true);
    expect(gotControllers.myOtherDirective instanceof MyOtherController)
      .toBe(true);
  });
});
```

要实现这个也是十分简单的。就像之前上一节，我们会对集合进行便利。这次我们要使用的是 LoDash 的`_.mapValues`函数，它会返回一个新对象，但新对象的 key 会跟原来一样，而值会被替换为映射函数（mapping function）的返回值。

_src/compile.js_

```js
function getControllers(require) {
  // if (_.isArray(require)) {
  //   return _.map(require, getControllers);
  } else if (_.isObject(require)) {
    return _.mapValues(require, getControllers);
  // } else {
  //   var value;
  //   if (controllers[require]) {
  //     value = controllers[require].instance;
  //   }
  //   if (!value) {
  //     throw 'Controller ' + require + ' required by directive, cannot be found!';
  //   }
  //   return value;
  // }
}
```

正如我们在上面的测试用例中看到的，其实语法中有重复的地方。也就是，对象的 key 和 value 都是要引入的指令名称。然而，这并不是必要的重复，因为 Angular 允许我们在 value 直接使用一个空字符串来代替（重复的指令名称）。

```js
it('can be required as an object with values omitted', function() {
  function MyController() {}
  var gotControllers;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: true,
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: {
          myDirective: '',
        },
        link: function(scope, element, attrs, controllers) {
          gotControllers = controllers;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive my-third-directive></div>');
    $compile(el)($rootScope);
    expect(gotControllers).toBeDefned();
    expect(gotControllers.myDirective instanceof MyController).toBe(true);
  });
});
```

在指令注册期间，我们会对省略的值进行填充。当我们实例化一个指令时，我们会使用一个新的帮助函数`getDirectiveRequire`来初始化指令的`require`属性：

_src/compile.js_

```js
$provide.factory(name + 'Directive', ['$injector', function($injector) {
  var factories = hasDirectives[name];
  return _.map(factories, function(factory, i) {
    // var directive = $injector.invoke(factory);
    // directive.restrict = directive.restrict || 'EA';
    // directive.priority = directive.priority || 0;
    // if (directive.link && !directive.compile) {
    //   directive.compile = _.constant(directive.link);
    // }
    // directive.$$bindings = parseDirectiveBindings(directive);
    // directive.name = directive.name || name;
    directive.require = getDirectiveRequire(directive);
    // directive.index = i;
    // return directive;
  });
}]);
```

这个函数会对`require`进行检查，看它是不是一个对象（并且不是数组），如果是则对它进行遍历，如果值为空，则使用对应的 key 的名称来进行填充：

```js
function getDirectiveRequire(directive) {
  var require = directive.require;
  if (!_.isArray(require) && _.isObject(require)) {
    _.forEach(require, function(value, key) {
      if (!value.length) {
        require[key] = key;
      }
    });
  }
  return require;
}
```

之后，我们还会对这个函数增加一些功能，因为我们需要对“引入祖先元素上的控制器”这个功能进行支持。
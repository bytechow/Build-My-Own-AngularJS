### ngController指令（The ngController Directive）

我们会以对`ngController`指令的讨论和开发来结束这一章，这很可能是所有 Angular 开发者都很熟悉的指令。举例来说，angularjs.org 网站上的第二个代码示例就是用这个指令的：

```html
<div ng-controller="TodoController">
  <!-- ... -->
</div>
```

要开始我们对 ngController 的讲述，我们先写一个测试用例来试一下用这个指令来绑定一个注册了的控制器构造函数，这个控制器也已经被实例化的。我们会把这个测试放到新的文件中，这个文件命名为`test/directives/ng_controller_spec.js`：

_test/directives/ng_controller_spec.js_

```js
'use strict';
var $ = require('jquery');
var publishExternalAPI = require('../../src/angular_public');
var createInjector = require('../../src/injector');
describe('ngController', function() {
  beforeEach(function() {
    delete window.angular;
    publishExternalAPI();
  });
  it('is instantiated during compilation & linking', function() {
    var instantiated;

    function MyController() {
      instantiated = true;
    }
    var injector = createInjector(['ng', function($controllerProvider) {
      $controllerProvider.register('MyController', MyController);
    }]);
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div ng-controller="MyController"></div>');
      $compile(el)($rootScope);
      expect(instantiated).toBe(true);
    });
  });
});
```

我们也会对控制器接收注入`$scope`、`$element`和`$attrs`这些参数的情况进行测试：

```js
it('may inject scope, element, and attrs', function() {
  var gotScope, gotElement, gotAttrs;
  function MyController($scope, $element, $attrs) {
    gotScope = $scope;
    gotElement = $element;
    gotAttrs = $attrs;
  }
  var injector = createInjector(['ng', function($controllerProvider) {
    $controllerProvider.register('MyController', MyController);
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div ng-controller="MyController"></div>');
    $compile(el)($rootScope);
    expect(gotScope).toBeDefned();
    expect(gotElement).toBeDefned();
    expect(gotAttrs).toBeDefned();
  });
});
```

竟然我们有了上面这一个测试用例，我们也会测试看控制器接收到的作用域是否来自上下文继承下来的作用域，这意味着`ngController`应该创建一个新的作用域：

```js
it('has an inherited scope', function() {
  var gotScope;

  function MyController($scope, $element, $attrs) {
    gotScope = $scope;
  }
  var injector = createInjector(['ng', function($controllerProvider) {
    $controllerProvider.register('MyController', MyController);
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div ng-controller="MyController"></div>');
    $compile(el)($rootScope);
    expect(gotScope).not.toBe($rootScope);
    expect(gotScope.$parent).toBe($rootScope);
    expect(Object.getPrototypeOf(gotScope)).toBe($rootScope);
  });
});
```

现在，我们要创建一些东西来通过这些单元测试。我们新建一个新文件叫`src/directives/ng_controller.js`。实际上要开发的代码是十分简单的。下面就是它的全部：

src/directives/ng_controller.js_

```js
'use strict'

var ngControllerDirective = function() {
  return {
    restrict: 'A',
    scope: true,
    controller: '@'
  };
};

module.exports = ngControllerDirective
```

要让测试通过，我们只需要在`angular_public.js`中把这个新指令作为`ng`模块的一部分：

```js
function publishExternalAPI() {
  // setupModuleLoader(window);
  
  // var ngModule = window.angular.module('ng', []);
  // ngModule.provider('$flter', require('./flter'));
  // ngModule.provider('$parse', require('./parse'));
  // ngModule.provider('$rootScope', require('./scope'));
  // ngModule.provider('$q', require('./q').$QProvider);
  // ngModule.provider('$$q', require('./q').$$QProvider);
  // ngModule.provider('$httpBackend', require('./http_backend'));
  // ngModule.provider('$http', require('./http').$HttpProvider);
  // ngModule.provider('$httpParamSerializer',
  //   require('./http').$HttpParamSerializerProvider);
  // ngModule.provider('$httpParamSerializerJQLike',
  //   require('./http').$HttpParamSerializerJQLikeProvider);
  // ngModule.provider('$compile', require('./compile'));
  // ngModule.provider('$controller', require('./controller'));
  ngModule.directive('ngController',
    require('./directives/ng_controller'));
}
```

`ngController`指令简单得会让人感到惊讶。那是因为`ngController`指令在 Angular 应用中被普遍使用了，以至于常常被人误以为是 Angular 框架的重要组成成分。实际上，`ngController`的功能开发都是基于`$controller`服务和`$compile`服务中对控制器的支持。
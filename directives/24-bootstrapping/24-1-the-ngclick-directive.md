### $ngClick 指令（The ngClick Directive）

Angular 实现了很多内建的指令，这些指令用于不同用途。如果你对它们中的任意一个特别感兴趣，学到这里其实就已经有充分的知识储备去看这些指令的源代码，看看里面是怎么实现的。大部分内建指令都是使用我们构建过的核心特性实现的，并用某种方式把它们与 DOM 结合起来。

我们现在要实现其中一个指令，就是`ngClick`指令。`ngClick`指令可以用于对点击事件添加表达式。这个指令会作为 Angular 内建的事件处理指令的例子。在我们编写实例应用时也会用上这个指令，使用它我们可以对 UI 添加一些简单的交互。

跟之前一样，我们也是从单元测试开始。我们会为`ngClikc`创建一个新的单元测试文件，然后把我们很熟悉的依赖注入引用加入进去。

_test/directives/ng_click_spec.js_

```js
'use strict';
var $ = require('jquery');
var publishExternalAPI = require('../../src/angular_public');
var createInjector = require('../../src/injector');

describe('ngClick', function() {

  var $compile, $rootScope;

  beforeEach(function() {
    delete window.angular;
    publishExternalAPI();
    var injector = createInjector(['ng']);
    $compile = injector.get('$compile');
    $rootScope = injector.get('$rootScope');
  }); 

});
```

`ngClick`会做的其中一项工作，也是我们首先要开发的，就是当点击事件发生后会启动一个 digest。我们可以通过`$watch`在根作用域加入一个 spy 函数。当在带有`ng-click`的元素上点击时，我们希望 watch 函数会被调用：

```js
it('starts a digest on click', function() {
  var watchSpy = jasmine.createSpy();
  $rootScope.$watch(watchSpy);

  var button = $('<button ng-click="doSomething()"></button>');
  $compile(button)($rootScope);
  
  button.click();
  expect(watchSpy).toHaveBeenCalled();
});
```

> 我们还没定义`doSomething()`函数，因此这个表达式什么不会做。回忆一下我们在实现表达式的解析时，我们允许在表达式中调用并不存在的函数。虽然调用失败了，但不会产生影响。

我们可以通过实现一个简单的`ngClick`指令来满足测试的要求。这个指令只能使用属性形态表示。当链接时，我们会对元素添加一个点击事件处理器。当点击该元素时，它会启动一个 digest：

_src/directives/ng_click.js_

```js
'use strict';
function ngClickDirective() {
  return {
    restrict: 'A',
    link: function(scope, element) {
      element.on('click', function() {
        scope.$apply();
      }); 
    }
  }; 
}

module.exports = ngClickDirective;
```

为了让编译器能够处理这个指令，我们依然需要在`ng`模块注册这个指令：

_src/angular_public.js_

```js
function publishExternalAPI() {
  setupModuleLoader(window);

  var ngModule = window.angular.module('ng', []);
  ngModule.provider('$filter', require('./filter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
  ngModule.provider('$q', require('./q').$QProvider);
  ngModule.provider('$$q', require('./q').$$QProvider);
  ngModule.provider('$httpBackend', require('./http_backend'));
  ngModule.provider('$http', require('./http').$HttpProvider);
  ngModule.provider('$httpParamSerializer',
    require('./http').$HttpParamSerializerProvider);
  ngModule.provider('$httpParamSerializerJQLike',
    require('./http').$HttpParamSerializerJQLikeProvider);
  ngModule.provider('$compile', require('./compile'));
  ngModule.provider('$controller', require('./controller').$ControllerProvider);
  ngModule.provider('$interpolate', require('./interpolate'));
  ngModule.directive('ngController',
    require('./directives/ng_controller'));
  ngModule.directive('ngTransclude',
    require('./directives/ng_transclude'));
  ngModule.directive('ngClick',
    require('./directives/ng_click'));
}
```

此时，第一个测试就通过了！

`ngClick` 大多用于在点击事件发生时对绑定的表达式进行求值，会对应用产生某种影响。我们可以通过使用一个调用了函数的表达式来验证一下，看点击事件发生时`ngClick`是否会调用这个函数：

_src/directives/ng_click.js_

```js
it('evaluates given expression on click', function() {
  $rootScope.doSomething = jasmine.createSpy();
  var button = $('<button ng-click="doSomething()"></button>');
  $compile(button)($rootScope);

  button.click();
  expect($rootScope.doSomething).toHaveBeenCalled();
});
```

这个实现起来也很简单。回忆一下本书第一部分的内容，`scope.$appy`可以接受一个表达式作为参数。传入这个参数后，程序会在当前 scope 的语境下对表达式进行求值。在`ngClick`中，我们可以从当前元素的`ng-click`属性拿到这个表达式。我们可以从`Attributes`对象中轻易地拿到这个值，然后把它传递给`$apply`：

```js
function ngClickDirective() {
  return {
    restrict: 'A',
    link: function(scope, element, attrs) {
      element.on('click', function() {
        scope.$apply(attrs.ngClick);
      });
    }
  };
}
```

我们快要搞定`ngClick`了，但还有一样东西没有完成，就是我们还没能让表达式可以通过`$event`访问到点击事件对象。这可以让应用代码检查这个事件对象的内容或者阻止事件的冒泡：

_test/directives/ng_click_spec.js_

```js
it('passes $event to expression', function() {
  $rootScope.doSomething = jasmine.createSpy();
  var button = $('<button ng-click="doSomething($event)"></button>');
  $compile(button)($rootScope);

  button.click();
  var evt = $rootScope.doSomething.calls.mostRecent().args[0];
  expect(evt).toBeDefined();
  expect(evt.type).toBe('click');
  expect(evt.target).toBeDefined();
});
```

我们可以使用之前在表达式解析器已经开发好的、用于加入本地变量的特性进行提供。我们可以把 DOM 事件对象赋值给本地变量`$event`。虽然`scope.$apply()`不允许传递本地变量，但`scope.$eval()`可以，我们可以在调用`$apply`之前先调用一次`scope.$eval()`：

```js
function ngClickDirective() {
  return {
    restrict: 'A',
    link: function(scope, element, attrs) {
      element.on('click', function(evt) {
        scope.$eval(attrs.ngClick, {$event: evt});
        scope.$apply();
      }); 
    }
  }; 
}
```

以上就是开发`ngClick`的过程了！这个指令实际上开发起来是非常简单的，毕竟它只是把一些我们之前实现的核心特性组合起来而已：它是以指令的形式构建的，使用了 scope 和表达式解析器特性来完成整个功能。

> 其实 AngularJS 本身在这一块还有一个优化点：每次点击的时候，程序都会将表达式字符串转化成一个函数。而实际上 AngularJS 框架会在编译阶段再使用`$parse`服务进行解析，然后只需要在每次点击时调用这个解析函数就好了。如果你想加入这个优化，去试试吧！
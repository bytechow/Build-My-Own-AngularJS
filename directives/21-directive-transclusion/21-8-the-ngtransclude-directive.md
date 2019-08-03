### ngTransclude 指令（The ngTransclude Directive）

当你第一次了解 transclusion 时，很有可能是在`ng-transclude`指令的一个引用中了解到的，这个指令可以在 transclusion 指令里的模板中使用，用于标记 transclude 内容需要插入到这个位置。使用了`ng-transclude`后你就可以不用自行调用函数，或者说根本不用在意这个函数。这是使用 transclusion 进行开发时的一种更为声明式的方式。

其实，这个指令完全可以使用本章介绍的知识进行实现，不需要依赖其他模块的代码进行实现。我们先通过一个测试集来把这个指令的行为确定下来。所以我们会先新建`test/directives/ng_transclude_spec.js`测试文件，然后加入测试集的初始代码。我们会加入基本的 Angular 安装代码，还会加入一个帮助函数，用于对给定的模板创建一个 transclusion 指令。

_test/directives/ng_transclude_spec.js_

```js
'use strict';

var $ = require('jquery');
var publishExternalAPI = require('../../src/angular_public');
var createInjector = require('../../src/injector');

describe('ngTransclude', function() {

  beforeEach(function() {
    delete window.angular;
    publishExternalAPI();
  });

  function createInjectorWithTranscluderTemplate(template) {
    return createInjector(['ng', function($compileProvider) {
      $compileProvider.directive('myTranscluder', function() {
        return {
          transclude: true,
          template: template
        };
      });
    }]);
  }
  
});
```

这能帮助我们快速定义一个测试。第一个测试，我们希望模板中的某个元素包含有`ng-transclude`属性的 transclusion 指令，能把 transclude 内容放入到这个元素内：

```js
it('transcludes the parent directive transclusion', function() {
  var injector = createInjectorWithTranscluderTemplate(
    '<div ng-transclude></div>'
  );
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder>Hello</div>');
    $compile(el)($rootScope);
    expect(el.find('> [ng-transclude]').html()).toEqual('Hello');
  });
});
```

其次，有`ng-transclude`属性的元素原有的内容应该被移除掉：

```js
it('empties existing contents', function() {
  var injector = createInjectorWithTranscluderTemplate(
    '<div ng-transclude>Existing contents</div>'
  );
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder>Hello</div>');
    $compile(el)($rootScope);
    expect(el.find('> [ng-transclude]').html()).toEqual('Hello');
  });
});
```

除了使用指令属性，我们也可以直接用`ng-transclude`作为元素：

```js
it('may be used as element', function() {
  var injector = createInjectorWithTranscluderTemplate(
    '<ng-transclude>Existing contents</ng-transclude>'
  );
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder>Hello</div>');
    $compile(el)($rootScope);
    expect(el.find('> ng-transclude').html()).toEqual('Hello');
  });
});
```

最后，我们也可以用 CSS 样式类的方式来使用`ng-transclude`：

```js
it('may be used as class', function() {
  var injector = createInjectorWithTranscluderTemplate(
    '<div class="ng-transclude">Existing contents</div>'
  );
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder>Hello</div>');
    $compile(el)($rootScope);
    expect(el.find('> .ng-transclude').html()).toEqual('Hello');
  });
});
```

这就是我们为`ng-transclude`指令建立的测试集。下面我们就开发代码来让这些测试通过。我们需要一个指令工厂函数，并把它放到`src/directives/ng_transclude.js`：

```js
'use strict';

var ngTranscludeDirective = function() {

  return {
    
  };

};

module.exports = ngTranscludeDirective;
```

我们也需要在`ng`模块中注册这个指令，这样当使用了这个指令时，编译器能找到它：

_src/angular_public.js_

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
  // ngModule.directive('ngController',
  //   require('./directives/ng_controller'));
  ngModule.directive('ngTransclude',
    require('./directives/ng_transclude'));
}
```

在最后，我们会把细节进行填充。从字面上来看，这种类型指令需要做的就是调用 transclusion 函数，并把接收到的 DOM 元素插入到这个指令所在的元素，同时也会把这个元素原有的内容清除掉：

_src/directives/ng_transclude.js_

```js
var ngTranscludeDirective = function() {
  
  return {
    restrict: 'EAC',
    link: function(scope, element, attrs, ctrl, transclude) {
      transclude(function(clone) {
        element.empty();
        element.append(clone);
      });
    }
  };

};
```

实际上，`ng-transclude`就像`ng-controller`一样，只是一个简单的指令，尽管我们可能会觉得它是一个"重要的"框架特性。这两个特性都是让我们更方便地使用指令编译器提供的核心特性。
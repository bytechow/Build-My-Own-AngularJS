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
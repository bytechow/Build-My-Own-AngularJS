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
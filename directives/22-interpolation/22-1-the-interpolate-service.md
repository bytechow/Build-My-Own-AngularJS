### $interpolate 服务（The $interpolate service）

大部分需要用到 interpolation 的功能都会通过`$interpolate`服务进行。这也是我们开始探索这个特性的最佳“场所”。

Angular 开发者很少会直接用到这个服务，因为它也集成到`$compile`服务里去了。在本章，我们将看到这种集成是如何生效的，但首先我们会先看看`$interpolate`能做哪些较低层次的活。

首先，我们需要先启动`$interpolate`服务，现在我们对启动方式已经很熟悉了。我们需要一个测试集，所以先加入进来。第一个关于`$interpolate`的测试会检查这个服务是否真的存在：

_test/interpolate_spec.js_

```js
'use strict';

var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');

describe('$interpolate', function() {

  beforeEach(function() {
    delete window.angular;
    publishExternalAPI();
  });

  it('should exist', function() {
    var injector = createInjector(['ng']);
    expect(injector.has('$interpolate')).toBe(true);
  });

});
```

就像其他的核心服务一张，我们通过一个 Provider 来创建`$interpolate`。这个 Provider 的返回值实际上也是一个函数，跟`$compile`一样。现在，它只需要返回一个函数体为空的函数即可：

_src/interpolate.js_

```js
'use strict';

function $InterpolateProvider() {

  this.$get = function() {
    
    function $interpolate() {
    }
    
    return $interpolate;
  };

}

module.exports = $InterpolateProvider;
```

在真正加入这个服务之前，我们依然需要在`ng`模块加入它：

_src/angular_public.js_

```js
function publishExternalAPI() {
  setupModuleLoader(window);

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
  // ngModule.provider('$controller', require('./controller'))
  ngModule.provider('$interpolate', require('./interpolate'));
  // ngModule.directive('ngController',
  //   require('./directives/ng_controller'));
  // ngModule.directive('ngTransclude',
  //   require('./directives/ng_transclude'));
}
```
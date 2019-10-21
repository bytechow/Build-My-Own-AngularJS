### Provider（The Providers）

像实现 $q 服务时一样，我们的第一步就是为 $http 设置 Provider。但在 $http 中我们需要两个 Provider，这是因为 AngularJS 中的 HTTP 通信也是分为两种—— $http 和 $httpBackend

两者的区别在于，$httpBackend 处理较低层次的 XMLHttpRequest 的工作，而 $http 处理高层次的面向用户的工作。从开发者的角度来说，他们在大多数情况下只使用 $http 就足够了，因为 $httpBackend 只会在 $http 内部被调用，所以这个区别对他们来说并没有太多意义。

而这样的划分，将会在我们需要选用其他 HTTP 通信方式的情况下起到作用。举例子来说，ngMock 服务就会重写 $httpBackend，这样可以伪造 HTTP 调用方便测试。

无论如何，我们都需要这两个服务在 ng 模块中被注册：

_test/angular_public_spec.js_

```js
it('sets up $http and $httpBackend', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$http')).toBe(true);
  expect(injector.has('$httpBackend')).toBe(true);
});
```

这两个服务各用一个文件存放。$httpBackend 会存放在 http_backend.js，通过一个`Provider`创建:

_src/http_backend.js_

```js
'use strict';

function $HttpBackendProvider() {

  this.$get = function() {
  
  };
}

module.exports = $HttpBackendProvider;
```

而 $http 服务则存放在 http.js 文件中，同时也会由一个`Provider`创建，这个`Provider`会先注入`$httpBackend`服务，虽然我们暂时还没用到它：

_src/http.js_

```js
'use strict';

function $HttpProvider() {
  
  this.$get = ['$httpBackend', function($httpBackend) {
  
  }];

}

module.exports = $HttpProvider;
```

现在，我们就可以在 ng 模块中注册这两个模块了：

_src/angular_public.js_

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  var ngModule = angular.module('ng', []);
  ngModule.provider('$flter', require('./flter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
  ngModule.provider('$q', require('./q').$QProvider);
  ngModule.provider('$$q', require('./q').$$QProvider);
  ngModule.provider('$httpBackend', require('./http_backend'));
  ngModule.provider('$http', require('./http'));
}
```
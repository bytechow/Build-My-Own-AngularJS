### $controller provider

我们还是应该从引入控制器对象开始。`$controller`就是专用于此目的的服务。这个服务和它的 provider 是我们要提前添加的东西。

`$controller`服务会作为`ng`模块的一部分，我们先对此进行测试：

_test/angular_public_spec.js_

```js
it('sets up $controller', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$controller')).toBe(true);
});
```

这个 provider 会被独立放到一个文件`src/controller`中，而它的建立方式就跟我们已经创建的其他 provider 一样，会有一个 provider 构造函数，里面会有一个`$get`方法，调用这个方法会返回一个`$controller`服务实例：

_src/controller.js_

```js
'use strict';

function $ControllerProvider() {

  this.$get = function() {

  };

}

module.exports = $ControllerProvider;
```

现在我们通过在`ng`模块中注册了`$controller`服务，就可以引用到这个 provider：

_src/angular_public.js_

```js
function publishExternalAPI(){
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
  // require('./http').$HttpParamSerializerProvider);
  // ngModule.provider('$httpParamSerializerJQLike',
  // require('./http').$HttpParamSerializerJQLikeProvider);
  // ngModule.provider('$compile', require('./compile'));
  ngModule.provider('$controller', require('./controller'));
}
```


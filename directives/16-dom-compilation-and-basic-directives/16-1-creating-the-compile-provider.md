### 创建 $compile Provider（Creating The $compile Provider）

指令编译器会使用一个名为`$compile`的函数。跟`$rootScope`、`$parse`、`$q`和`$http`一样，`$compile`也是 Angular 内建的一个组件，它也可以通过依赖注入进行注入。当我们创建一个包含`ng`模块的注射器后，我们会希望`$compile`服务已经包含在这个注射器中：

_test/angular_public_spec.js_

```js
it('sets up $compile', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$compile')).toBe(true);
});
```

`$compile`服务会作为一个 provider 定义在`compile.js`这个新文件中：

_src/compile.js_

```js
'use strict';

function $CompileProvider() {

  this.$get = function() {
  
  };

}

module.exports = $CompileProvider;
```

我们会在`angular_public.js`中，对`ng`模块加入`$compile`服务，方式与我们之前加入过的服务一致：

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
  ngModule.provider('$http', require('./http').$HttpProvider);
  ngModule.provider('$httpParamSerializer',
    require('./http').$HttpParamSerializerProvider);
  ngModule.provider('$httpParamSerializerJQLike',
    require('./http').$HttpParamSerializerJQLikeProvider);
  ngModule.provider('$compile', require('./compile'));
}
```
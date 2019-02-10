## 拦截器（Interceptors）

前面的章节，我们介绍了如何请求和响应数据的转换器（transform），介绍了它们是如何对请求数据或响应数据进行修改，当然这里说的修改大部分都是用于序列化。现在，我们会介绍另一种会被应用于 HTTP 请求或响应的、更为人熟知、更为通用的特性——拦截器（Interceptors）。

相对于转换器，拦截器更为高级，功能也更为全面，确实适用于对 HTTP 请求和响应加入各种处理逻辑。使用拦截器，你可以自由地对 HTTP 请求、HTTP 响应进行修改或替换。由于拦截器本身就是基于 Promise 的，你可以在拦截器加入异步处理逻辑，这是转换器无法比拟的。

拦截器是使用工厂函数（factory function）创建的。要注册一个拦截器，你需要把这个拦截器的工厂函数加入到`$httpProvider`的`interceptors`数组。这意味着拦截器必须要在应用的配置阶段进行注册。一旦`$http`服务被创建，所有注册的拦截器函数将会被激活：

_test/http_spec.js_

```js
it('allows attaching interceptor factories', function() {
  var interceptorFactorySpy = jasmine.createSpy();
  var injector = createInjector(['ng', function($httpProvider) {
    $httpProvider.interceptors.push(interceptorFactorySpy);
  }]);
  $http = injector.get('$http');
  
  expect(interceptorFactorySpy).toHaveBeenCalled();
});
```
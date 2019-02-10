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

我们会在`$HttpProvider`构造函数中加入这个数组，并挂载在`interceptors`属性：

_src/http.js_

```js
function $HttpProvider() {
  var interceptorFactories = this.interceptors = [];
  // ...
}
```

当`$http`服务被创建时（也就是`$httpProvider.$get`被调用之时），我们就会调用所有注册进来的工厂函数，这些工厂函数都存放在刚才我们加入的数组中：

```js
this.$get = ['$httpBackend', '$q', '$rootScope', '$injector'
  function($httpBackend, $q, $rootScope, $injector) {
    var interceptors = _.map(interceptorFactories, function(fn) {
      return fn();
    });
    // ...    
  }
];
```

拦截器工厂函数还可以被集成到依赖注入的系统中。如果工厂函数时带有参数的，那这些参数对应的依赖将会被注入。当然我们还可以使用行内式依赖注入的方式，比如`['a', 'b', function(a, b){ }]`。下面我们来测试一下注入`$rootScope`：

_test/http_spec.js_

```js
it('uses DI to instantiate interceptors', function() {
  var interceptorFactorySpy = jasmine.createSpy();
  var injector = createInjector(['ng', function($httpProvider) {
    $httpProvider.interceptors.push(['$rootScope', interceptorFactorySpy]);
  }]);
  $http = injector.get('$http');
  var $rootScope = injector.get('$rootScope');
  expect(interceptorFactorySpy).toHaveBeenCalledWith($rootScope);
});
```

我们会使用注射器的`invoke`方法来实例化所有拦截器，而非直接调用：

_src/http.js_

```js
// this.$get = ['$httpBackend', '$q', '$rootScope', '$injector',
//   function($httpBackend, $q, $rootScope, $injector) {
//     var interceptors = _.map(interceptorFactories, function(fn) {
      return $injector.invoke(fn);
//     });
//     // ...
//   }
// ];
```

目前，我们通过把拦截器函数加入到`$httpProvider.interceptors`数组来注册拦截器，但其实还有一种方法。我们可以先注册一个普通的 Angular 工厂函数，然后把它的名称加入到`$httpProvider.interceptors`中去。这种方式其实才是在 AngularJS 语境中的最优解法——“拦截器也只是一个工厂函数”：

_test/http_spec.js_

```js
it('allows referencing existing interceptor factories', function() {
  var interceptorFactorySpy = jasmine.createSpy().and.returnValue({});
  var injector = createInjector(['ng', function($provide, $httpProvider) {
    $provide.factory('myInterceptor', interceptorFactorySpy);
    $httpProvider.interceptors.push('myInterceptor');
  }]);
  $http = injector.get('$http');
  expect(interceptorFactorySpy).toHaveBeenCalled();
});
```

在创建拦截器时，我们必须对传入的拦截器元素进行判断，看它是用函数还是字符串来进行注册的。如果是字符串，我们就要使用`$inject.get`（该方法实际上会调用工厂方法）方法获取对应的拦截器函数；如果是函数，那我们就像之前一样直接调用即可：

_src/http.js_

```js
var interceptors = _.map(interceptorFactories, function(fn) {
  return _.isString(fn) ? $injector.get(fn) :
    $injector.invoke(fn);
});
```


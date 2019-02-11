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

现在我们知道了拦截器是如何注册的，我们可以正式来聊聊它们究竟是什么，还有它们是如何被集成到`$http`服务的请求中去。

拦截器十分依赖 Promise。它们会作为一个 Promise 回调函数加入到`$http`的处理流程中，同时它们自身也会返回 Promise。当前的`$http`已经集成了 Promise（因为它会返回一个 Promise），但在我们集成拦截器进去之前，我们需要对代码进行重新组织。

首先，之前存在于`$http`函数体内的代码，其中一部分需要放到拦截器之前执行，而另一部分需要放在拦截器之后执行。而需要在拦截器之后执行的代码自然就需要放到一个 Promise 的回调函数中去执行，我们需要先把这段代码逻辑抽取出来放到一个新的函数中，我们将这个函数命名为`serverRequest`。目前我们不会改动这段代码逻辑，只需要把它们迁移到新函数就好了。我们会把分隔点放到 config 对象变量创建、头部也被加入到 config 变量之后：

```js
function serverRequest(confg) {
  if (_.isUndefned(confg.withCredentials) &&
    !_.isUndefned(defaults.withCredentials)) {
    confg.withCredentials = defaults.withCredentials;
  }
  var reqData = transformData(
    confg.data,
    headersGetter(confg.headers),
    undefned,
    confg.transformRequest
  );
  if (_.isUndefned(reqData)) {
    _.forEach(confg.headers, function(v, k) {
      if (k.toLowerCase() === 'content-type') {
        delete confg.headers[k];
      }
    });
  }

  function transformResponse(response) {
    if (response.data) {
      response.data = transformData(
        response.data,
        response.headers,
        response.status,
        confg.transformResponse
      );
    }
    if (isSuccess(response.status)) {
      return response;
    } else {
      return $q.reject(response);
    }
  }
  return sendReq(confg, reqData)
    .then(transformResponse, transformResponse);
}

// function $http(requestConfg) {
//   var confg = _.extend({
//     method: 'GET',
//     transformRequest: defaults.transformRequest,
//     transformResponse: defaults.transformResponse,
//     paramSerializer: defaults.paramSerializer
//   }, requestConfg);
//   if (_.isString(confg.paramSerializer)) {
//     confg.paramSerializer = $injector.get(confg.paramSerializer);
//   }
//   confg.headers = mergeHeaders(requestConfg);
  
  return serverRequest(confg);
// }
```

下一步，我们要改变一下创建`$http` Promise 的方式。之前，我们直接把`sendReq`返回的 Promise 作为最终返回值，但现在我们会基于`$http`函数里的`config`对象变量创建一个 Promise，然后把`serverRequest`作为这个 Promise 的回调处理函。此举并不会改变`$http`的处理逻辑，但却能为我们加入拦截器打下基础：

```js
function $http(requestConfg) {
  var confg = _.extend({
    method: 'GET',
    transformRequest: defaults.transformRequest,
    transformResponse: defaults.transformResponse,
    paramSerializer: defaults.paramSerializer
  }, requestConfg);
  if (_.isString(confg.paramSerializer)) {
    confg.paramSerializer = $injector.get(confg.paramSerializer);
  }
  confg.headers = mergeHeaders(requestConfg);
  
  var promise = $q.when(confg);
  return promise.then(serverRequest);
}
```

虽然我们还没有真正改变`$http`的处理逻辑，但你会发现这个改动已经让很多单元测试都无                            法通过了。这是因为如果我们现在发送请求，这个请求只会在调用`$http`函数后的下一个 digest 循环中才会被实际发送出去。原因是现在`serverRequest`只会在 Promise 回调函数中被调用，而 Promise 回到函数只会在 digest 循环中被实际执行。

这就是 Angular 的特性，我们能做的就是要对这些发生异常的单元测试进行调整。其中最主要的就是我们需要在每个已经存在的`    `
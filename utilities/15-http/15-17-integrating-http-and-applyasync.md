## 集成 $applyAsync 到 $http（Integrating $http and $applyAsync）

在本章的最后，我们会讨论一下对`$http`的一个重要优化。

在本书的第一章，我们实现了 Scopes，也说到了`$applyAsync`特性是在 Angular 1.3 版本才有的。它最主要的用处就是让我们可以延后执行一个函数，这个延后也只是“稍稍”延后。这个功能可以让我们把最近一段时间的 digest 循环要干的事都整合起来，延后到某个时间点再统一执行——如果在短时间内内有多个`$applyAsync`调用，那它们会被放到同一个 digest 循环完成。

其实`$applyAsync`的诞生就是为`$http`服务。一个应用启动后，马上会在短时间内发送多个请求到服务器，以获取不同的资源，这是非常常见的。如果服务器的响应足够快，我们很有可能在短时间内面对要处理多个响应的情况。按照目前的代码实现，在这种情况下，我们需要对每个响应都启动一个 digest 循环。

我们要做的优化，就是用`$applyAsync`来控制开启一个处理服务器响应的 digest 循环。如果多个响应在短时间内到达，我们就会在同一个 digest 循环在处理应用数据的变更。这很可能会给应用带来极为重要的性能优化，尤其是进入应用后首次加载资源时，优化程度视应用程序的不同而有差异罢了。

但`$applyAsync`优化默认是不开启的。你必须在应用配置阶段显式地调用`$httpProvider.useApplyAsync(true)`来启用。我们会在单元测试的`beforeEach`代码块加入这一步：

_test/http_spec.js_

```js
describe('useApplyAsync', function() {

  beforeEach(function() {
    var injector = createInjector(['ng', function($httpProvider) {
      $httpProvider.useApplyAsync(true);
    }]);
    $http = injector.get('$http');
    $rootScope = injector.get('$rootScope');
  });
  
});
```

当启用这个优化之后，我们期望响应回调函数不会在响应到达后马上就被调用。这一点就跟我们之前的不同：

```js
it('does not resolve promise immediately when enabled', function() {
  var resolvedSpy = jasmine.createSpy();
  $http.get('http://teropa.info').then(resolvedSpy);
  $rootScope.$apply();
  
  requests[0].respond(200, {}, 'OK');
  expect(resolvedSpy).not.toHaveBeenCalled();
});
```

相反，响应回调会在一段时间过去后被触发：

```js
it('resolves promise later when enabled', function() {
  var resolvedSpy = jasmine.createSpy();
  $http.get('http://teropa.info').then(resolvedSpy);
  $rootScope.$apply();

  requests[0].respond(200, {}, 'OK');
  jasmine.clock().tick(100);
  
  expect(resolvedSpy).toHaveBeenCalled();
});
```

在`$httpProvider`中，我们会设置`useApplyAsync`方法和一个同名的标志变量。这个方法可以有两种调用方式：要么带上参数调用，要么不带参数调用，如果不带上参数，它还会返回当前的标识变量`useApplyAsync`的值：

_src/http.js_

```js
function $HttpProvider() {
  var interceptorFactories = this.interceptors = [];

  var useApplyAsync = false;
  this.useApplyAsync = function(value) {
    if (_.isUndefned(value)) {
      return useApplyAsync;
    } else {
      useApplyAsync = !!value;
      return this;
    }
  };

  // ...
  
}
```

在我们的`done`处理函数中，我们需要把 resolve Promise 的代码放到一个辅助函数`resolvePromise`中。根据`useApplyAsync`标识变量的值，我们会选择是立即执行这个函数（也就是使用`$rootScope.$apply`直接调用），还是稍后才执行这个函数（使用`$rootScope.$applyAsync`）。

```js
// function done(status, response, headersString, statusText) {
//   status = Math.max(status, 0);

  function resolvePromise() {
    // deferred[isSuccess(status) ? 'resolve' : 'reject']({
    //   status: status,
    //   data: response,
    //   statusText: statusText,
    //   headers: headersGetter(headersString),
    //   confg: confg
    // });
  }
  
  if (useApplyAsync) {
    $rootScope.$applyAsync(resolvePromise);
  } else {
    resolvePromise();
    // if (!$rootScope.$$phase) {
    //   $rootScope.$apply();
    // }
  }
// }
```
## Promise 扩展（Promise

如果你使用过 Angular 的 `$http`，你可能会发现它返回的 Promise 还会包含一些方法，而这些方法目前我们还没有实现。实际上也就是`success`和`error`两个方法，而且这两个方法只会出现在`$http`返回的 Promise 上。

这个扩展的出现，主要是为了更好地处理 HTTP 响应。使用`then`或`catch`处理块，我们获取到的参数值是完整的响应对象。而`success`会传入的是四个已经拆分好的参数：响应数据、状态码、头部和原始请求配置：

_test/http_spec.js_

```js
it('allows attaching success handlers', function() {
  var data, status, headers, confg;
  $http.get('http://teropa.info').success(function(d, s, h, c) {
    data = d;
    status = s;
    headers = h;
    confg = c;
  });
  $rootScope.$apply();

  requests[0].respond(200, {
    'Cache-Control': 'no-cache'
  }, 'Hello');
  $rootScope.$apply();
  
  expect(data).toBe('Hello');
  expect(status).toBe(200);
  expect(headers('Cache-Control')).toBe('no-cache');
  expect(confg.method).toBe('GET');
});
```

虽然这个不算是极为重要的特性，但能够给我们带来便利。尤其是当你只需要获取响应体数据时，我们就可以使用一个`success`处理函数，这个函数只需要接收一个参数，这个参数就是响应体数据了，我们不需要在乎响应对象的格式。

如果是错误响应，我们有一个同样的扩展，那就是`error`回调方法：

```js
it('allows attaching error handlers', function() {
  var data, status, headers, confg;
  $http.get('http://teropa.info').error(function(d, s, h, c) {
    data = d;
    status = s;
    headers = h;
    confg = c;
  });
  $rootScope.$apply();
  
  requests[0].respond(401, {
    'Cache-Control': 'no-cache'
  }, 'Fail');
  $rootScope.$apply();

  expect(data).toBe('Fail');
  expect(status).toBe(401);
  expect(headers('Cache-Control')).toBe('no-cache');
  expect(confg.method).toBe('GET');
});
```

在我们把所有的拦截都绑定好了以后，这两个扩展的方法会被加入到`Promise`对象中。在它们内部会使用`then`或`catch`回调，回调中会将拆分开的参数依次传递到要执行的回调函数中，这个回调函数就是上面单元测试中`success`或`error`方法中传入的函数类型的参数：

```js
var promise = $q.when(confg);
_.forEach(interceptors, function(interceptor) {
  promise = promise.then(interceptor.request, interceptor.requestError);
});
promise = promise.then(serverRequest);
_.forEachRight(interceptors, function(interceptor) {
  promise = promise.then(interceptor.response, interceptor.responseError);
});
promise.success = function(fn) {
  promise.then(function(response) {
    fn(response.data, response.status, response.headers, confg);
  });
  return promise;
};
promise.error = function(fn) {
  promise.catch(function(response) {
    fn(response.data, response.status, response.headers, confg);
  });
  return promise;
};
return promise;
```
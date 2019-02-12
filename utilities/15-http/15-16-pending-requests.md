## 进行中的请求（Pending Requests）

如果能够在请求中获取实时信息，在特定情况下会很有用，比如调试，但也有可能有其他的用途。

这个特性可以借助拦截器来完成，但其实在`$http`中已经有内建特性进行支持了。任何正在进行中的请求实际上都会放到`$http.pendingRequests`这个可访问到的数组中。每发送一个请求，我们就会往数组里面添加这个请求对象，如果接收到对应的响应（无论它是成功响应还是失败响应），我们就会把这个请求对象从数组中移除：

_test/http_spec.js_

```js
describe('pending requests', function() {
  
  it('are in the collection while pending', function() {
    $http.get('http://teropa.info');
    $rootScope.$apply();
    
    expect($http.pendingRequests).toBeDefned();
    expect($http.pendingRequests.length).toBe(1);
    expect($http.pendingRequests[0].url).toBe('http://teropa.info');
    requests[0].respond(200, {}, 'OK');
    $rootScope.$apply();

    expect($http.pendingRequests.length).toBe(0);
  });

  it('are also cleared on failure', function() {
    $http.get('http://teropa.info');
    $rootScope.$apply();

    requests[0].respond(404, {}, 'Not found');
    $rootScope.$apply();

    expect($http.pendingRequests.length).toBe(0);
  });
});
```

在`$httpProvider.get`方法中，我们会初始化这个数组，并把它赋值给`$http`对象：

_src/http.js_

```js
$http.defaults = defaults;
$http.pendingRequests = [];
```

接下来，我们就可以根据请求的状态变更在`sendReq`方法中对这个数组进行添加或删除的操作。在请求发送出去时，我们会往数组中加入这个请求对应的配置对象；而在请求的 Promise 的状态变成已解决（可能是 resolved，也可能是 rejected）时，我们会把请求对象从数组中移除：

```js
function sendReq(confg, reqData) {
  var deferred = $q.defer();
  $http.pendingRequests.push(confg);
  deferred.promise.then(function() {
    _.remove($http.pendingRequests, confg);
  }, function() {
    _.remove($http.pendingRequests, confg);
  });

  // ...
}
```

要注意的是，因为我们是在`sendReq`方法中加入这个功能，而不是在`$http`中，所以当请求对象还存在于拦截器处理阶段时，该请求不能算是“进行中”（pending）。只有当请求真的被发送出去后，才能算是“进行中”（pending）。
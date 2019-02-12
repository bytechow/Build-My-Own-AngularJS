## 请求超时（Request Timeouts）

网络请求有时可能会花费很长的时间，有可能是服务器应答耗费了很长时间，也可能是网络出了问题导致服务器应答无法到达客户端。

浏览器、服务器和网络代理本身已经有一套内建的机制来应对网络超时，但对于应用开发者来说，我们也可能需要自定义超时时长。Angular 也提供了一些特性来满足这种需求。

首先，我们可以直接在请求配置对象里面加入`timeout`属性，我们会传入一个 Promise 作为这个属性的值。如果 Promise 在响应返回之前被 resolve 了，Angular 会自动中断请求。

这个特性确实非常强大，因为我们可以根据实际需要来控制请求次数。举一个例子，如果用户要跳转到另一个路由，或关闭对话框，这时我们就可以主动告知应用代码，从而不需要等待一个不再会被使用到的响应。

_test/http\_spec.js_

```js
it('allows aborting a request with a Promise', function() {
  var timeout = $q.defer();
  $http.get('http://teropa.info', {
    timeout: timeout.promise
  });
  $rootScope.$apply();

  timeout.resolve();
  $rootScope.$apply();

  expect(requests[0].aborted).toBe(true);
});
```

当然，要正常运行这个单元测试，我们需要引入`$q`服务：

```js
var $http, $rootScope, $q;
// var xhr, requests;

// beforeEach(function() {
//   publishExternalAPI();
//   var injector = createInjector(['ng']);
//   $http = injector.get('$http');
//   $rootScope = injector.get('$rootScope');
  $q = injector.get('$q');
// });
```

超时管理的实际工作会在 HTTP 底层完成。在`$http`中我们需要做的，就是把请求配置参数中的`timeout`属性传递进去：

_src/http\_backend.js_

```js
// $httpBackend(
//   confg.method,
//   url,
//   reqData,
//   done,
//   confg.headers,
  confg.timeout,
//   confg.withCredentials
// );
```

在`$httpBackend`我们会检查是否接收到`timeout`参数，如果有，则调用 XHLHttpRequest 自带的`abort`方法中断请求：

```js
return function(method, url, post, callback, headers, timeout, withCredentials) {
  // var xhr = new window.XMLHttpRequest();
  // xhr.open(method, url, true);
  // _.forEach(headers, function(value, key) {
  //   xhr.setRequestHeader(key, value);
  // });
  // if (withCredentials) {
  //   xhr.withCredentials = true;
  // }
  // xhr.send(post || null);
  // xhr.onload = function() {
  //   var response = ('response' in xhr) ? xhr.response :
  //     xhr.responseText;
  //   var statusText = xhr.statusText || '';
  //   callback(
  //     xhr.status,
  //     response,
  //     xhr.getAllResponseHeaders(),
  //     statusText
  //   );
  // };
  // xhr.onerror = function() {
  //   callback(-1, null, '');
  // };
  if (timeout) {
    timeout.then(function() {
      xhr.abort();
    });
  }
};
```

除了这种基于 Promise 的实现，我们也可以用数字作为`timeout`的属性值，表示超过了多长的时间后 Angular 会自动中断请求过程。

这时，我们需要用到 Jasmine 的 clock API 来模拟 JavaScript 内部的运行时钟（JavaScript's internal clock）。这需要我们在每次单元测试之前先注册 clock，当单元测试测试结束后，我们需要销毁掉这个时钟：

_test/http\_spec.js_

```js
beforeEach(function() {
  jasmine.clock().install();
});
afterEach(function() {
  jasmine.clock().uninstall();
});
```

现在我们可以看到如果在请求配置对象中指定一段等待时间，当等待时间过去后，请求会被终止掉：

```js
it('allows aborting a request after a timeout', function() {
  $http.get('http://teropa.info', {
    timeout: 5000
  });
  $rootScope.$apply();

  jasmine.clock().tick(5001);

  expect(requests[0].aborted).toBe(true);
});
```

在`$httpBackend`里面，我们发现传入了一个数字类型的`timeout`参数，而不是 Promise 类型的参数，我们就会使用原生的`setTimeout` API 来中止请求：

_src/http\_backend.js_

```js
if (timeout && timeout.then) {
  timeout.then(function() {
    xhr.abort();
  });
} else if (timeout > 0) {
  setTimeout(function() {
    xhr.abort();
  }, timeout);
}
```

当然，我们需要保证如果请求在指定的等待时间之前完成了，则不需要中断请求，所以在这种情况下，我们需要清除掉定时器：

```js
// return function(method, url, post, callback, headers, timeout, withCredentials) {
//   var xhr = new window.XMLHttpRequest();
  var timeoutId;
  // xhr.open(method, url, true);
  // _.forEach(headers, function(value, key) {
  //   xhr.setRequestHeader(key, value);
  // });
  // if (withCredentials) {
  //   xhr.withCredentials = true;
  // }
  // xhr.send(post || null);
  // xhr.onload = function() {
    if (!_.isUndefned(timeoutId)) {
      clearTimeout(timeoutId);
    }
  //   var response = ('response' in xhr) ? xhr.response :
  //     xhr.responseText;
  //   var statusText = xhr.statusText || '';
  //   callback(
  //     xhr.status,
  //     response,
  //     xhr.getAllResponseHeaders(),
  //     statusText
  //   );
  // };
  // xhr.onerror = function() {
    if (!_.isUndefned(timeoutId)) {
      clearTimeout(timeoutId);
    }
  //   callback(-1, null, '');
  // };
  // if (timeout && timeout.then) {
  //   timeout.then(function() {
  //     xhr.abort();
  //   });
  // } else if (timeout > 0) {
    timeoutId = setTimeout(function() {
//       xhr.abort();
//     }, timeout);
//   }
// };
```




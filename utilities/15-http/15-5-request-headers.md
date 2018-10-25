### 请求头（Request Headers）

当发送请求到 HTTP 服务器时，加上请求头是很重要的。请求头包含了我们需要告知服务器的各种信息，如授权 token、希望获取的内容类型和 HTTP 缓存控制。

$http 服务完全支持 HTTP 请求头部，我们可以在请求配置对象中加入一个 headers 对象属性：

_test/http_spec.js_

```js
it('sets headers on request', function() {
  $http({
    url: 'http://teropa.info',
    headers: {
      'Accept': 'text/plain',
      'Cache-Control': 'no-cache'
    }
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders.Accept).toBe('text/plain');
  expect(requests[0].requestHeaders['Cache-Control']).toBe('no-cache');
});
```

对于实现请求头部，`$httpBackend`服务会承担主要的工作。而在`$http`我们就只需要传递 headers 对象过去即可：

_src/http.js_

```js
return function $http(requestConfg) {
  // ...
  
  $httpBackend(
    confg.method,
    confg.url,
    confg.data,
    done,
    confg.headers
  );
  return deferred.promise;
};
```

`$httpBackend`则需要对 headers 进行遍历，并通过 XHR 对象的`setRequestHeader`方法来设置头部：

_src/http_backend.js_

```js
var _ = require('lodash');

function $HttpBackendProvider() {

  this.$get = function() {
    return function(method, url, post, callback, headers) {
      var xhr = new window.XMLHttpRequest();
      xhr.open(method, url, true);
      _.forEach(headers, function(value, key) {
        xhr.setRequestHeader(key, value);
      });
      xhr.send(post || null);
      // ...
    };
  };
}

module.exports = $HttpBackendProvider;
```

即使没有传入 headers，请求头部也会有一些默认配置。其中最重要的，就是`Accept`请求头部。这个头部用来告诉服务器，客户端优先接收 JSON类型作为响应数据，其次是纯文本的响应：

```js
it('sets default headers on request', function() {
  $http({
  url: 'http://teropa.info'
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders.Accept).toBe('application/json, text/plain, */*');
});
```
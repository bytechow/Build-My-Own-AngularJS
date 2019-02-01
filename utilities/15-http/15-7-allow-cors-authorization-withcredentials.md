## 允许CORS授权：withCredentials（Allow CORS Authorization: withCredentials）

如果我们想发送 XHR 请求去访问其他域的资源时，就会遇到浏览器安全性限制。目前我们主要通过 cross-origin resource sharing（CORS，跨域资源分享）来对这些限制进行管理。

与 CORS 相关的工作大多不需要通过 JavaScript 来完成：它更多需要通过服务器和浏览器之间的通信完成。但如果想让我们的`$http`服务完全支持 CORS，则我们必须要做一些必要的工作。跨域请求默认不包括任何 cookies 或授权头部。如果一个跨域请求需要两者之一，就需要对我们的 XHR 对象设置`withCredentials`标识。

在 Angular 里面，你可以将`withCredentials`标识放到请求对象中去：

_test/http_spec.js_

```js
it('allows setting withCredentials', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42,
    withCredentials: true
  });
  expect(requests[0].withCredentials).toBe(true);
});
```

这个标识会在`$http`中被提取，并在调用`$httpBackend`时作为一个参数传入：

_src/http.js_

```js
$httpBackend(
  confg.method,
  confg.url,
  confg.data,
  done,
  confg.headers,
  confg.withCredentials
);
```

在 backend，如果传入了该标识，那我们就也为 XHR 对象增加这个标识：

_src/http_backend.js_

```js
return function(method, url, post, callback, headers, withCredentials) {
  var xhr = new window.XMLHttpRequest();
  xhr.open(method, url, true);
  _.forEach(headers, function(value, key) {
    xhr.setRequestHeader(key, value);
  });
  if (withCredentials) {
    xhr.withCredentials = true;
  }
  xhr.send(post || null);
  // ...
};
```

我们也可以通过`defaults`在全局配置这个参数：

_test/http_spec.js_

```js
it('allows setting withCredentials from defaults', function() {
  $http.defaults.withCredentials = true;
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  });
  expect(requests[0].withCredentials).toBe(true);
});
```

在`$http`中构建请求配置时，当且仅当设置了默认的`withCredentials`为真值（truthy），并且当前请求没有显式传入`withCredentials`参数时，才会将默认设置变成当前请求的设置：

_src/http.js_

```js
function $http(requestConfg) {
  var deferred = $q.defer();
  var confg = _.extend({
    method: 'GET'
  }, requestConfg);
  confg.headers = mergeHeaders(requestConfg);
  if (_.isUndefned(confg.withCredentials) &&
    !_.isUndefned(defaults.withCredentials)) {
    confg.withCredentials = defaults.withCredentials;
  }
  // ...
}
```
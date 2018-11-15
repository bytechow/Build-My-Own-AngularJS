## 响应头部（response headers）

现在已经可以通过几种方式设置请求头部，那我们可以把目光转向另一个头部——响应头部。

Angular 支持所有可以从 HTTP 服务器返回的响应头部。它们都将会被放在`$http`服务中的响应对象（response object）之内。响应对象将会有一个 headers 属性指向一个针对头部的访问函数。这个函数就会接收头部名称，并返回对应的头部值：

```js
it('makes response headers available', function() {
  var response;
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  }).then(function(r) {
    response = r;
  });

  requests[0].respond(200, {
    'Content-Type': 'text/plain'
  }, 'Hello');
  
  expect(response.headers).toBeDefned();
  expect(response.headers instanceof Function).toBe(true);
  expect(response.headers('Content-Type')).toBe('text/plain');
  expect(response.headers('content-type')).toBe('text/plain');
});
```


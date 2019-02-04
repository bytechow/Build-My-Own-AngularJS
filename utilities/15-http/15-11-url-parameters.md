## URL 参数（URL Parameters）

到目前为止，我们已经看到`$http`服务发送请求信息的三个部分：请求 URL、请求头部和请求体。接下来我们会讲解最后一种能往 HTTP 请求中加入信息的方式——URL 查询参数（URL _query parameters_）.实际上，也就是在 URL 中问号后面的名值对。

诚然，我们目前的代码已经支持使用查询参数了，毕竟我们可以把查询参数当作 URL 的一部分进行访问。但这样的形式对于参数序列化或跟踪管理上还是比较麻烦，所以可能更希望 Angular 帮你完成这部分的工作。事实上在 Angular 中，我们可以在请求配置中使用`params`属性完成传递 URL 参数的任务：

_test/http_spec.js_

```js
it('adds params to URL', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: 42
    }
  });
  
  expect(requests[0].url).toBe('http://teropa.info?a=42');
});
```

我们还应该对已经含有 query 参数的 URL 进行适配，在这种情况下我们应该把`params`属性传递的参数继续往后拼接上：

```js
it('adds additional params to URL', function() {
  $http({
    url: 'http://teropa.info?a=42',
    params: {
      b: 42
    }
  });
  
  expect(requests[0].url).toBe('http://teropa.info?a=42&b=42');
});
```

在`$http`服务里面，在我们发送请求到 HTTP 后台之前，我们会利用两个辅助函数来完成 URL 的构建，这两个函数分别叫`serializeParams`和`buildUrl`。前者会接受请求参数，并把它们序列化一个 URL 字符串，后者会把请求 URL 和参数字符串拼合在一起：

_src/http.js_

```js
var url = buildUrl(confg.url, serializeParams(confg.params));

// $httpBackend(
//   confg.method,
  url,
//   reqData,
//   done,
//   confg.headers,
//   confg.withCredentials
// );
```
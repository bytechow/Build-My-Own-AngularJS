## 便捷方法（Shorthand Methods）

到目前为止，我们已经完成了`$http`服务关于处理 HTTP 响应的全部内容，接下来我们要关注的是`$http`提供给应用开发者的相关 API，然后我们会用 Promise 异步工作流来处理请求。

其中，`$http`本身会提供一些便捷方法来让请求过程变得更流式化（streamlined）。比如，我们要为`$http`新增一个`get`方法，这个方法用于接收一个请求的 URL 和一个请求配置对象（可选），最终组成一个 GET 请求：

_test/http_spec.js_

```js
it('supports shorthand method for GET', function() {
  $http.get('http://teropa.info', {
    params: {q: 42}
  });

  expect(requests[0].url).toBe('http://teropa.info?q=42');
  expect(requests[0].method).toBe('GET');
});
```

事实上，这个便捷方法在底层还是会调用`$http`函数，而且要保证配置对象包含给定的 URL 和 GET 方法：

```js

```
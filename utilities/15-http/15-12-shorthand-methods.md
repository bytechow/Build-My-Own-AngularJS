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

_src/http.js_

```js
// $http.defaults = defaults;
$http.get = function(url, confg) {
  return $http(_.extend(confg || {}, {
    method: 'GET',
    url: url
  }));
};
// return $http;
```

我们也支持 HEAD 和 DELETE 请求，这两种请求与 GET 请求很类似：

_test/http_spec.js_

```js
it('supports shorthand method for HEAD', function() {
  $http.head('http://teropa.info', {
    params: {
      q: 42
    }
  });
  expect(requests[0].url).toBe('http://teropa.info?q=42');
  expect(requests[0].method).toBe('HEAD');
});

it('supports shorthand method for DELETE', function() {
  $http.delete('http://teropa.info', {
    params: {
      q: 42
    }
  });
  expect(requests[0].url).toBe('http://teropa.info?q=42');
  expect(requests[0].method).toBe('DELETE');
});
```

这两种请求的便捷方法除了使用的 HTTP 方法是不同的，其余的代码实现跟 GET 请求是一模一样的，：

_src/http.js_

```js
$http.head = function(url, confg) {
  return $http(_.extend(confg || {}, {
    method: 'HEAD',
    url: url
  }));
};
$http.delete = function(url, confg) {
  return $http(_.extend(confg || {}, {
    method: 'DELETE',
    url: url
  }));
};
```

实际上，我们会通过一个循环来完成这三个方法的构建，这样可以规避重复代码的出现：

```js
// $http.defaults = defaults;
_.forEach(['get', 'head', 'delete'], function(method) {
  $http[method] = function(url, confg) {
    return $http(_.extend(confg || {}, {
      method: method.toUpperCase(),
      url: url
    }));
  };
});
// return $http;
```

`$http`还会提供另外三个 HTTP 方法——POST、PUT 和 PATCH。这三个方法与前面三个的区别在于，这三个方法都支持请求体（request body）,也就是说我们可以对请求附加`data`属性。所以这次我们要实现的便捷方法将会支持传递三个参数：URL、请求体数据（可选）、请求配置参数（可选）：

_test/http_spec.js_

```js
it('supports shorthand method for POST with data', function() {
  $http.post('http://teropa.info', 'data', {
    params: {
      q: 42
    }
  });
  
  expect(requests[0].url).toBe('http://teropa.info?q=42');
  expect(requests[0].method).toBe('POST');
  expect(requests[0].requestBody).toBe('data');
});

it('supports shorthand method for PUT with data', function() {
  $http.put('http://teropa.info', 'data', {
    params: {
      q: 42
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?q=42');
  expect(requests[0].method).toBe('PUT');
  expect(requests[0].requestBody).toBe('data');
});

it('supports shorthand method for PATCH with data', function() {
  $http.patch('http://teropa.info', 'data', {
    params: {
      q: 42
    }
  });
  
  expect(requests[0].url).toBe('http://teropa.info?q=42');
  expect(requests[0].method).toBe('PATCH');
  expect(requests[0].requestBody).toBe('data');
});
```

我们在生命 GET、HEAD 和 DELETE 三个便捷方法的循环下面，加入另一个循环来生成后面三个便捷方法：

```js
_.forEach(['post', 'put', 'patch'], function(method) {
  $http[method] = function(url, data, confg) {
    return $http(_.extend(confg || {}, {
      method: method.toUpperCase(),
      url: url,
      data: data
    }));
  };
});
```
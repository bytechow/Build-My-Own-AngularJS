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

让我们来编写这两个函数。`serializeParams`函数会遍历参数对象，并把里面的每一组参数变成字符串，这个字符串以等号（=）分割参数名和参数值。待遍历完毕，最后再把所有参数组字符串都用`&`连接在一起：

```js
function serializeParams(params) {
  var parts = [];
  _.forEach(params, function(value, key) {
    parts.push(key + '=' + value);
  });
  return parts.join('&');
}
```

`buildUrl`就单纯是将序列化好的参数字符串拼接在原有的 URL 上即可。要注意的是，它会根据传入的 URL 是否包含`?`，来决定是用`?`还是`&`作为连接的字符：

```js
function buildUrl(url, serializedParams) {
  if (serializedParams.length) {
    url += (url.indexOf('?') === -1) ? '?' : '&';
    url += serializedParams;
  }
  return url;
}
```

URL 参数可能会含有一些特殊字符，无法直接拼接到 URL 上。比如`=`和`&`，显然很容易会与参数分隔符产生混淆。因此，我们需要先对特殊字符进行转义：

_test/http_spec.js_

```js
it('escapes url characters in params', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      '==': '&&'
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?%3D%3D=%26%26');
});
```

JavaScript 原生支持的`encodeURIComponent`就可以用于转义：

_src/http.js_

```js
// function serializeParams(params) {
//   var parts = [];
//   _.forEach(params, function(value, key) {
//     parts.push(
      encodeURIComponent(key) + '=' + encodeURIComponent(value));
//   });
//   return parts.join('&');
// }
```

> 实际上，AngularJS 并不会直接使用`encodeURIComponent`来进行转码，而是采用内建的工具方法——`encodeUriQuery`。这个方法的转义范围并不会像`encodeURIComponent`那样广，它不会对`@`和`:`进行转义。

如果传入请求配置对象的参数值是`null`和`undefined`，这个参数就会被忽略掉：

_test/http_spec.js_

```js
it('does not attach null or undefned params', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: null,
      b: undefned
    }
  });
  expect(requests[0].url).toBe('http://teropa.info');
});
```

我们可以在便利时加入一个检测，如果值为空则跳过：

```js
function serializeParams(params) {
  var parts = [];
  _.forEach(params, function(value, key) {
    if (_.isNull(value) || _.isUndefned(value)) {
      return;
    }
    parts.push(
      encodeURIComponent(key) + '=' + encodeURIComponent(value));
  });
  return parts.join('&');
}
```

HTTP 协议支持使用同一个参数名传递多个值。这块可以通过重用同一个参数名来完成，Angular 也支持这种传参方式，我们会使用数组的形式来存储这类同名参数值：

_test/http_spec.js_

```js
it('attaches multiple params from arrays', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: [42, 43]
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?a=42&a=43');
});
```

为此，我们需要在每次参数遍历时，在内部也进行一次内部的遍历。为了兼容所有类型的参数，我们把所有非数组类型的参数都包裹在数组里：

_src/http.js_

```js
// function serializeParams(params) {
//   var parts = [];
//   _.forEach(params, function(value, key) {
//     if (_.isNull(value) || _.isUndefned(value)) {
//       return;
//     }
    if (!_.isArray(value)) {
      value = [value];
    }
    _.forEach(value, function(v) {
      parts.push(
        encodeURIComponent(key) + '=' + encodeURIComponent(v));
    });
//   });
//   return parts.join('&');
// }
```

因此，数组用作参数值时是有特殊用途的。那对象呢？HTTP 协议默认并不支持“嵌套参数”（nested parameter）,那如果我们传入对象作为参数值会发生什么呢（之后需要进行 URL 转义）。事实上，如果服务器已经准备好对 JSON 数据进行解析，我们也可以在 query 参数中传递：

_test/http_spec.js_

```js
it('serializes objects to json', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: { b: 42 }
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?a=%7B%22b%22%3A42%7D');
});
```

在我们对参数值进行遍历时，我们需要检测这个值是不是对象类型，若是需要将其转成 JSON 数据：

_src/http.js_

```js
// function serializeParams(params) {
//   var parts = [];
//   _.forEach(params, function(value, key) {
//     if (_.isNull(value) || _.isUndefned(value)) {
//       return;
//     }
//     if (!_.isArray(value)) {
//       value = [value];
//     }
//     _.forEach(value, function(v) {
      if (_.isObject(v)) {
        v = JSON.stringify(v);
      }
//       parts.push(
//         url += encodeURIComponent(key) + '=' + encodeURIComponent(v));
//     });
//   });
//   return parts.join('&');
// }
```

要做 JSON 序列化，就需要考虑对日期的处理，在序列化（JSON.stringify）的过程中，Date 类型的值会自动转变为 ISO 8601 格式。为了验证这个事实，我们会多增加一个单元测试：

_test/http_spec.js_

```js
it(‘serializes dates to ISO strings’, function() {
  $http({
    url: ‘http: //teropa.info’,
      params: {
        a: new Date(2015, 0, 1, 12, 0, 0)
      }
  });
  $rootScope.$apply();
  
  expect(/\d{4}-\d{2}-\d{2}T\d{2}%3A\d{2}%3A\d{2}/
    .test(requests[0].url)).toBeTruthy();
});
```

确实，我们对日期类型的 URL 参数进行序列化，默认的结果就是会输出为 ISO 8601 格式。而对于 Angular 应用开发者来说，你也可以使用其他序列化方法来代替。你可以在请求配置对象或`$http`默认配置中添加一个`paramSerializer`属性，它是一个数组，可以把传入的对象参数转化为字符串
## 请求数据转换（Request Transforms）

当与服务器通信时，我们经常需要把请求数据进行预处理，从而将数据转换为服务器能理解的数据格式，如 JSON、XML，或其他自定义格式。

在使用 Angular 时，当然你也是完全可以单独对每一个要发送的请求进行处理：确保每一个放在 data 属性中的数据都能被服务器理解。但要重复进行这样的处理也是不够理想的。如果我们可以把预处理从业务代码中抽取出来，这样会更便利。这就是我们需要`request transforms`(请求数据转换器)的原因。

请求数据转换器是一个函数，它会在我们发送请求体之前，对请求体进行预处理。处理后的对象将会替代原来的请求体数据。

> 我们不应该混淆转换器与拦截器，在本章的后面部分我们会对此进行解释。

绑定请求转换器的其中一个方法，就是在请求对象中新增`transformRequest`属性：

_test/http_spec.js_

```js
it('allows transforming requests with functions', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42,
    transformRequest: function(data) {
      return '*' + data + '*';
    }
  });
  expect(requests[0].requestBody).toBe('*42*');
});
```

我们在`$http`中会通过在发送请求之前调用`transformData`函数来应用转换器。这个函数有两个参数，一个是请求数据对象，另一个是 transformRequest 属性值。它的返回值最终会成为实际的请求数据：

_src/http.js_(未被注释的代码为新增代码)

```js
function $http(requestConfg) {
  // var deferred = $q.defer();
  // var confg = _.extend({
  //   method: 'GET'
  // }, requestConfg);
  // confg.headers = mergeHeaders(requestConfg);
  // if (_.isUndefned(confg.withCredentials) &&
  //   !_.isUndefned(defaults.withCredentials)) {
  //   confg.withCredentials = defaults.withCredentials;
  // }
  var reqData = transformData(confg.data, confg.transformRequest);
  if (_.isUndefned(reqData)) {
  //   _.forEach(confg.headers, function(v, k) {
  //     if (k.toLowerCase() === 'content-type') {
  //       delete confg.headers[k];
  //     }
  //   });
  // }

  // function done(status, response, headersString, statusText) {
  //   status = Math.max(status, 0);
  //   deferred[isSuccess(status) ? 'resolve' : 'reject']({
  //     status: status,
  //     data: response,
  //     statusText: statusText,
  //     headers: headersGetter(headersString),
  //     confg: confg
  //   });
  //   if (!$rootScope.$$phase) {
  //     $rootScope.$apply();
  //   }
  // }
  // $httpBackend(
  //   confg.method,
  //   confg.url,
  //   reqData,
  //   done,
  //   confg.headers,
  //   confg.withCredentials
  // );
  // return deferred.promise;
}
```

仅当转换器函数存在时，`transformData`函数才会执行转换，否则它仅仅是返回原始的请求数据：

```js
function transformData(data, transform) {
  if (_.isFunction(transform)) {
    return transform(data);
  } else {
    return data;
  }
}
```

Angular 也支持传递多个请求转换器，只需要把`transformRequest`的属性值变成一个函数数组即可，它们会被依次执行：

_test/http_spec.js_

```js
it('allows multiple request transform functions', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42,
    transformRequest: [function(data) {
      return '*' + data + '*';
    }, function(data) {
      return '-' + data + '-';
    }]
  });

  expect(requests[0].requestBody).toBe('-*42*-');
});
```

我们会使用 _reduce_ 的方式来实现对属性形式的`transformData`的处理。我们也会利用 \_.reduce 会在传入的 transform 参数为空时返回原始数据的属性：

```
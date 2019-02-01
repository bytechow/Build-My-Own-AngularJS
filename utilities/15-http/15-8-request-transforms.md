## 请求数据转换（Request Transforms）

当与服务器通信时，我们经常需要把请求数据进行预处理，从而将数据转换为服务器能理解的数据格式，如 JSON、XML，或其他自定义格式。

在使用 Angular 时，当然你也是完全可以单独对每一个要发送的请求进行处理：确保每一个放在 data 属性中的数据都能被服务器理解。但要重复进行这样的处理也是不够理想的。如果我们可以把预处理从业务代码中抽取出来，这样会更便利。这就是我们需要`request transforms`\(请求数据转换器\)的原因。

请求数据转换器是一个函数，它会在我们发送请求体之前，对请求体进行预处理。处理后的对象将会替代原来的请求体数据。

> 我们不应该混淆转换器与拦截器，在本章的后面部分我们会对此进行解释。

绑定请求转换器的其中一个方法，就是在请求对象中新增`transformRequest`属性：

_test/http\_spec.js_

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

_src/http.js_\(未被注释的代码为新增代码\)

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

_test/http\_spec.js_

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

我们会使用 _reduce_ 的方式来实现对属性形式的`transformData`的处理。我们也会利用`_.reduce`会在传入的 transform 参数为空时返回原始数据的特性：

_src/http.js_

```js
// function transformData(data, transform) {
//   if (_.isFunction(transform)) {
//     return transform(data);
//   } else {
    return _.reduce(transform, function(data, fn) {
      return fn(data);
    }, data);
//   }
// }
```

对每一个请求对象配置`transformRequest`固然好用，但配置默认属性可能更为常用。如果你对`$http.defaults`配置了转换器函数，就意味着“让每个请求在发送之前都执行这个转换”，这实现了更高层次的关注点分离——这使得你在发送请求时无需再考虑请求转换：

_test/http\_spec.js_

```js
it('allows settings transforms in defaults', function() {
  $http.defaults.transformRequest = [function(data) {
    return '*' + data + '*';
  }];
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  });
  expect(requests[0].requestBody).toBe('*42*');
});
```

我们可以在构建请求配置对象时插入默认的`transformRequest`：

_src/http.js_

```js
function $http(requestConfg) {
  var deferred = $q.defer();

  var confg = _.extend({
    method: 'GET',
    transformRequest: defaults.transformRequest
  }, requestConfg);
  confg.headers = mergeHeaders(requestConfg);

  // ...
}
```

对于默认请求转换器来说，除了请求体，它们可能还会需要一些额外信息来完成它们的工作——例如，有些转换器可能只会在某些特定的 HTTP 内容类型下才会被执行。出于这方面的考虑，我们将请求头部作为转换器函数的第二个参数，这个请求头部参数将会被封装为一个根据传入的头部名称返回头部值的函数：

_test/http\_spec.js_

```js
it('passes request headers getter to transforms', function() {
  $http.defaults.transformRequest = [function(data, headers) {
    if (headers('Content-Type') === 'text/emphasized') {
      return '*' + data + '*';
    } else {
      return data;
    }
  }];
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42,
    headers: {
      'content-type': 'text/emphasized'
    }
  });

  expect(requests[0].requestBody).toBe('*42*');
});
```

因此我们需要把请求头部传递给`transformData`。在此之前，我们需要将请求头部先通过`headersGetter`进行处理，因为这样会生成一个符合我们预期的用于获取头部的函数。当然，目前`headersGetter`目前还不能处理对象类型的请求头部对象，但我们马上就会修复这个问题。

_src/http.js_

```js
// var reqData = transformData(
  // confg.data,
  headersGetter(confg.headers),
  // confg.transformRequest
// );
```

现在，在`transformData`函数中，我们就可以把获取到的请求头部传递到各个实际的转换器函数中去了：

_src/http.js_

```js
function transformData(data, headers, transform) {
  // if (_.isFunction(transform)) {
    return transform(data, headers);
  // } else {
    // return _.reduce(transform, function(data, fn) {
      return fn(data, headers);
    // }, data);
  // }
}
```

要完成这个功能，我们还差一步。我们得让`headersGetter`，更准确来说，是让`parseHeaders`函数学会如何处理我们传入的请求头部对象。

之前我们已经在`parseHeaders`实现了将请求头部字符串转换成为对象的功能。而对于请求头部对象，它本身就是一个对象了，所以我们只需要做“标准化”即可，也就是让请求头部名变成小写，并去除头部名和头部值两侧的空格：

_src/http.js_

```js
// function parseHeaders(headers) {
  if (_.isObject(headers)) {
    return _.transform(headers, function(result, v, k) {
      result[_.trim(k.toLowerCase())] = _.trim(v);
    }, {});
  } else {
    // var lines = headers.split('\n');
    // return _.transform(lines, function(result, line) {
    //   var separatorAt = line.indexOf(':');
    //   var name = _.trim(line.substr(0, separatorAt)).toLowerCase();
    //   var value = _.trim(line.substr(separatorAt + 1));
    //   if (name) {
    //     result[name] = value;
    //   }
    // }, {});
  }
// }
```




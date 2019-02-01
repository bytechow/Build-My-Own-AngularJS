## 响应数据转换（Response Transforms）

类似请求数据转换，如果我们能在响应数据从服务器返回后、到达应用代码前进行预处理，这将变得很便利。一个典型的应用场景是，我们可以将响应对象中的序列化数据（JSON）变成 JavaScript 对象。

响应数据的转换，与请求数据转换是对称的。你可以在请求配置对象中加入`transformResponse`属性，它将会在获取到响应数据体时调用：

_test/http_spec.js_

```js
it('allows transforming responses with functions', function() {
  var response;
  $http({
    url: 'http://teropa.info',
    transformResponse: function(data) {
      return '*' + data + '*';
    }
  }).then(function(r){ 
    response = r;
  }); 

  requests[0].respond(200, {'Content-Type': 'text/plain'}, 'Hello'); 

  expect(response.data).toEqual('*Hello*');
});
```

与请求转换器一样，响应转换器函数也提供了头部对象作为第二个参数，只不过这个头部对象是响应头部，而不是请求头部：

```js
it('passes response headers to transform functions', function() {
  var response;
  $http({
    url: 'http://teropa.info',
    transformResponse: function(data, headers) {
      if (headers('content-type') === 'text/decorated') {
        return '*' + data + '*';
      } else {
        return data;
      }
    }
  }).then(function(r) {
    response = r;
  });

  requests[0].respond(200, {
    'Content-Type': 'text/decorated'
  }, 'Hello');
  
  expect(response.data).toEqual('*Hello*');
});
```

另外，我们也可以在`$http`服务配置默认的响应数据转换器，这样你就不用在每个请求配置对象中都配置：

```js
it('allows setting default response transforms', function() {
  $http.defaults.transformResponse = [function(data) {
    return '*' + data + '*';
  }];
  var response;
  $http({
    url: 'http://teropa.info'
  }).then(function(r) {
    response = r;
  });

  requests[0].respond(200, {
    'Content-Type': 'text/plain'
  }, 'Hello');
  
  expect(response.data).toEqual('*Hello*');
});
```

在让这些单元测试都变“绿”（通过）之前，我们会先对`$http`服务中代码进行重新组织。`$http`函数本身已经变得很长了，我们会将它划分为两个步骤：准备请求，然后发送请求。在`$http`函数中，我们会继续保留准备请求的代码，而把发送请求的代码逻辑搬迁到一个新的函数`sendReq`：

_src/http.js_

```js
function sendReq(confg, reqData) {
  var deferred = $q.defer();

  function done(status, response, headersString, statusText) {
    status = Math.max(status, 0);
    deferred[isSuccess(status) ? 'resolve' : 'reject']({
      status: status,
      data: response,
      statusText: statusText,
      headers: headersGetter(headersString),
      confg: confg
    });
    if (!$rootScope.$$phase) {
      $rootScope.$apply();
    }
  }
  
  $httpBackend(
    confg.method,
    confg.url,
    reqData,
    done,
    confg.headers,
    confg.withCredentials
  );

  return deferred.promise;
}

function $http(requestConfg) {
  var confg = _.extend({
    method: 'GET',
    transformRequest: defaults.transformRequest
  }, requestConfg);
  confg.headers = mergeHeaders(requestConfg);

  if (_.isUndefned(confg.withCredentials) &&
    !_.isUndefned(defaults.withCredentials)) {
    confg.withCredentials = defaults.withCredentials;
  }

  var reqData = transformData(
    confg.data,
    headersGetter(confg.headers),
    confg.transformRequest
  );

  if (_.isUndefned(reqData)) {
    _.forEach(confg.headers, function(v, k) {
      if (k.toLowerCase() === 'content-type') {
        delete confg.headers[k];
      }
    });
  }

  return sendReq(confg, reqData);
}
```

整理过后，我们就更容易把转换响应对象的代码放进去了。现在，我们会新建一个函数来专门用于转换对象，它将会作为`sendReq`的 promise 回调：

```js
// function $http(requestConfg) {
//   var confg = _.extend({
//     method: 'GET',
//     transformRequest: defaults.transformRequest,
//   }, requestConfg);
//   confg.headers = mergeHeaders(requestConfg);
  
//   if (_.isUndefned(confg.withCredentials) &&
//     !_.isUndefned(defaults.withCredentials)) {
//     confg.withCredentials = defaults.withCredentials;
//   }

//   var reqData = transformData(
//     confg.data,
//     headersGetter(confg.headers),
//     confg.transformRequest
//   );

//   if (_.isUndefned(reqData)) {
//     _.forEach(confg.headers, function(v, k) {
//       if (k.toLowerCase() === 'content-type') {
//         delete confg.headers[k];
//       }
//     });
//   }

  function transformResponse(response) {}

  // return sendReq(confg, reqData)
    .then(transformResponse);
// }
```

这个函数会接受一个响应对象作为参数，并把响应对象的`data`属性赋值为转换结果。我们可以重用`transformData`函数：

```js
function transformResponse(response) {
  if (response.data) {
    response.data = transformData(response.data, response.headers,
      confg.transformResponse);
  }
  return response;
}
```

我们也要加入对默认响应数据转换器的支持：

```js
function $http(requestConfg) {
  var confg = _.extend({
    method: 'GET',
    transformRequest: defaults.transformRequest,
    transformResponse: defaults.transformResponse
  }, requestConfg);
  
  // ...
}
```

我们不仅要支持对成功响应对象的支持，同时也要支持失败响应对象的支持，因为失败响应对象也可能会存在需要转换的数据：

_test/http_spec.js_

```js
it('transforms error responses also', function() {
  var response;
  $http({
    url: 'http://teropa.info',
    transformResponse: function(data) {
      return '*' + data + '*';
    }
  }).catch(function(r) {
    response = r;
  });

  requests[0].respond(401, {
    'Content-Type': 'text/plain'
  }, 'Fail');
  
  expect(response.data).toEqual('*Fail*');
});
```

`transformResponse`函数实际上已经支持这种情况，但由于它被用作一个promise的回调，我们需要再次 reject 请求失败响应。否则，请求失败会被认为是已被“捕获”，错误响应结果将会在成功回调中被接收，我们需要对此进行修复：

_src/http.js_

```js
// function transformResponse(response) {
//   if (response.data) {
//     response.data = transformData(response.data, response.headers,
//       confg.transformResponse);
//   }
  if (isSuccess(response.status)) {
    // return response;
  } else {
    return $q.reject(response);
  }
// }
// return sendReq(confg, reqData)
  .then(transformResponse, transformResponse);
```

响应数据转换器的最后一个特性，也是请求数据转换器不具备的一个特性：响应对象的状态码（status code）。它将会作为每一个响应数据转换器的第三个参数。

_test/http_spec.js_

```js
it('passes HTTP status to response transformers', function() {
  var response;
  $http({
    url: 'http://teropa.info',
    transformResponse: function(data, headers, status) {
      if (status === 401) {
        return 'unauthorized';
      } else {
        return data;
      }
    }
  }).catch(function(r) {
    response = r;
  });

  requests[0].respond(401, {
    'Content-Type': 'text/plain'
  }, 'Fail');
  
  expect(response.data).toEqual('unauthorized');
});
```

在`transformResponse`，我们可以将状态码提取出来，并传递给`transformData`：

_src/http.js_

```js
// function transformResponse(response) {
//   if (response.data) {
//     response.data = transformData(
//       response.data,
//       response.headers,
      response.status,
//       confg.transformResponse
//     );
//   }
//   if (isSuccess(response.status)) {
//     return response;
//   } else {
//     return $q.reject(response);
//   }
// }
```

上面我们加入了一个额外的参数给`transformData`，但由于请求数据转换也是使用改函数进行转换，所以对于请求数据转换，我们需要在调用时显式地传入`undefined`作为`transformData`的第三个参数——请求并不需要状态码：

```js
// var reqData = transformData(
//   confg.data,
//   headersGetter(confg.headers),
  undefned,
//   confg.transformRequest
// );
```

在`transformData`我们可以接收这个参数，并将其传递到具体的转换器函数：

```js
function transformData(data, headers, status, transform) {
  // if (_.isFunction(transform)) {
    return transform(data, headers, status);
  // } else {
  //   return _.reduce(transform, function(data, fn) {
      return fn(data, headers, status);
//     }, data);
//   }
// }
```
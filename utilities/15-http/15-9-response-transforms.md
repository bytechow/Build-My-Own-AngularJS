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
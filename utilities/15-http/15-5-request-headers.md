### 请求头（Request Headers）

当发送请求到 HTTP 服务器时，加上请求头是很重要的。请求头包含了我们需要告知服务器的各种信息，如授权 token、希望获取的内容类型和 HTTP 缓存控制。

$http 服务完全支持 HTTP 请求头部，我们可以在请求配置对象中加入一个 headers 对象属性：

_test/http_spec.js_

```js
it('sets headers on request', function() {
  $http({
    url: 'http://teropa.info',
    headers: {
      'Accept': 'text/plain',
      'Cache-Control': 'no-cache'
    }
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders.Accept).toBe('text/plain');
  expect(requests[0].requestHeaders['Cache-Control']).toBe('no-cache');
});
```

对于实现请求头部，`$httpBackend`服务会承担主要的工作。而在`$http`我们就只需要传递 headers 对象过去即可：

_src/http.js_

```js
return function $http(requestConfg) {
  // ...
  
  $httpBackend(
    confg.method,
    confg.url,
    confg.data,
    done,
    confg.headers
  );
  return deferred.promise;
};
```

`$httpBackend`则需要对 headers 进行遍历，并通过 XHR 对象的`setRequestHeader`方法来设置头部：

_src/http_backend.js_

```js
var _ = require('lodash');

function $HttpBackendProvider() {

  this.$get = function() {
    return function(method, url, post, callback, headers) {
      var xhr = new window.XMLHttpRequest();
      xhr.open(method, url, true);
      _.forEach(headers, function(value, key) {
        xhr.setRequestHeader(key, value);
      });
      xhr.send(post || null);
      // ...
    };
  };
}

module.exports = $HttpBackendProvider;
```

即使没有传入 headers，请求头部也会有一些默认配置。其中最重要的，就是`Accept`请求头部。这个头部用来告诉服务器，客户端优先接收 JSON类型作为响应数据，其次是纯文本的响应：

```js
it('sets default headers on request', function() {
  $http({
  url: 'http://teropa.info'
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders.Accept).toBe('application/json, text/plain, */*');
});
```

我们把这些默认的请求头部都放到`$HttpProvider`构造函数的一个对象变量`defaults`中去，在这个对象变量里面我们新增一个`headers`属性，在`headers`里面再新增`common`属性，这代表它保存了各个 HTTP 方法中通用的请求头部。Accept 头部就是其中之一：

_src/http.js_

```js
function $HttpProvider() {
  
  var defaults = {
    headers: {
      common: {
        Accept: 'application/json, text/plain, */*'
      }
    }
  };

  // ...

}
```

现在，我们需要在请求配置对象中合并这些默认配置，这项工作会放到一个新的帮助函数`mergeHeaders`中完成：

_src/http.js_

```js
return function $http(requestConfg) {
  var deferred = $q.defer()

  var confg = _.extend({
    method: 'GET'
  }, requestConfg)

  confg.headers = mergeHeaders(requestConfg)
  
  // ...
    
};
```

现在，这个函数会先创建一个新的对象，这个对象会依次合并（merge）`default`的通用请求头和通过请求配置参数传入的请求头：

```js
function mergeHeaders(confg) {
  return _.extend(
    {},
    defaults.headers.common,
    confg.headers
  );
}
```

不是所有的默认请求头都适用于各个 HTTP 请求方法。比如说，POST 方法应该有一个默认的头部`Content-Type`，其值为 JSON 类型，但 GET 方法就不需要这个默认限制。这是因为 GET 请求并没有请求体（body），所以设定它们的内容类型时不合适的。

_test/http_spec.js_

```js
it('sets method-specifc default headers on request', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: '42'
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders['Content-Type']).toBe(
  'application/json;charset=utf-8');
});
```

defaults 变量也会包含每个请求方法特有的请求头部。下面我们会把 POST、PUT 和 PATCH 请求方法的`Conten-Type`都设置为 JSON 类型：

```js
var defaults = {
  headers: {
    common: {
      Accept: 'application/json, text/plain, */*'
    },
    post: {
      'Content-Type': 'application/json;charset=utf-8'
    },
    put: {
      'Content-Type': 'application/json;charset=utf-8'
    },
    patch: {
      'Content-Type': 'application/json;charset=utf-8'
    }
  }
};
```

现在我们会在`mergeHeaders`帮助函数的返回值中加入这些仅限于某些请求方法的默认头部：

```js
function mergeHeaders(confg) {
  return _.extend({},
    defaults.headers.common,
    defaults.headers[(confg.method || 'get').toLowerCase()],
    confg.headers
  );
}
```

对于应用开发者来说，他们也可以对默认请求头进行修改。具体来说，可以直接通过`$http`服务提供的接口直接覆写`defaults`属性：

_test/http_spec.js_

```js
it('exposes default headers for overriding', function() {
  $http.defaults.headers.post['Content-Type'] = 'text/plain;charset=utf-8';
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: '42'
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders['Content-Type']).toBe(
    'text/plain;charset=utf-8');
});
```

_src/http.js_

```js
function $http(requestConfg) {
  var deferred = $q.defer();
  var confg = _.extend({
    method: 'GET'
  }, requestConfg);
  confg.headers = mergeHeaders(requestConfg);

  function done(status, response, statusText) {
    status = Math.max(status, 0);
    deferred[isSuccess(status) ? 'resolve' : 'reject']({
      status: status,
      data: response,
      statusText: statusText,
      confg: confg
    });
    if (!$rootScope.$$phase) {
      $rootScope.$apply();
    }
  }
  $httpBackend(confg.method, confg.url, confg.data, done, confg.headers);
  return deferred.promise;
}
$http.defaults = defaults;
return $http;
```
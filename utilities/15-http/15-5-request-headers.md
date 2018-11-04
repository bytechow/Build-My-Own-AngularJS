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

这些请求头部不仅可以在运行时通过`$http`进行修改，还可以通过`$httpProvider`进行配置。我们可以通过使用自定义的函数模块来创建对`$httpProvider`的注入和配置：

_test/http_spec.js_

```js
it('exposes default headers through provider', function() {
  var injector = createInjector(['ng', function($httpProvider) {
    $httpProvider.defaults.headers.post['Content-Type'] =
      'text/plain;charset=utf-8';
  }]);
  $http = injector.get('$http');
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

我们可以直接把 default 绑定到`$HttpProvider`上的`this`上去：

_src/http.js_

```js
function $HttpProvider() {
  var defaults = this.defaults = {
    //...
  };
  
  //...
}
```

> 我们现在可以在两处地方对默认请求头进行配置：defaults 存储变量中的默认头部配置，和运行时创建的`$http`对象中的默认请求方法。两者的不同在于，前者是允许进行修改，而后者不允许。默认的请求方法在应用中无法修改

如果你熟悉 HTTP 请求头部的运作，你就会知道它们的名称实际上是不区分大小写的：`Content-Type`和`content-type`会被视为同一个配置。显然，当前我们的实现还没支持这一特性：我们很可能会使用不同的大小写形式来表达同一个配置。我们应该要支持不区分大小写的请求头配置：

```js
it('merges default headers case-insensitively', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: '42',
    headers: {
      'content-type': 'text/plain;charset=utf-8'
    }
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders['content-type']).toBe(
    'text/plain;charset=utf-8');
  expect(requests[0].requestHeaders['Content-Type']).toBeUndefned();
});
```

这意味着，我们得改变`mergeHeaders`的运行方式。我们首先会将头部分为两种：一种叫`reqHeaders`,用于存放请求配置对象中传入的headers；而另一种叫`defHeaders`用于存放所有默认头部——包含了通用的和指定了请求方法的头部：

```js
function mergeHeaders(confg) {
  var reqHeaders = _.extend({},
    confg.headers
  );
  var defHeaders = _.extend({},
    defaults.headers.common,
    defaults.headers[(confg.method || 'get').toLowerCase()]
  );
}
```

现在，我们需要把这两个对象进行合并。我们会把`reqHeaders`作为一个起点，并把`defHeaders`中的属性合并进去。对于每一个默认头部，我们都会检测是不是已经有同名属性，并进行大小写检测。如果当前结果对象中没有这个属性才会被加入进去：

_src/http.js_

```js
function mergeHeaders(confg) {
  var reqHeaders = _.extend({},
    confg.headers
  );
  var defHeaders = _.extend({},
    defaults.headers.common,
    defaults.headers[(confg.method || 'get').toLowerCase()]
  );
  _.forEach(defHeaders, function(value, key) {
    var headerExists = _.some(reqHeaders, function(v, k) {
      return k.toLowerCase() === key.toLowerCase();
    });
    if (!headerExists) {
      reqHeaders[key] = value;
    }
  });
  return reqHeaders;
}
```

上面我们处理了头部合并，现在我们可以关注另一些特殊情况。

我们已经看到`Content-Type`请求头部被默认设置为 JSON 类型。但我们应该保证当请求配置中没有请求体时，`Content-Type`不会被加入，毕竟没有请求内容，也没有必要说明请求的内容类型。所以，如果没有请求体，我们需要忽略掉`Content-Type`，即使它已经配置了：

```js
it('does not send content-type header when no data', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    headers: {
      'Content-Type': 'application/json;charset=utf-8'
    }
  });
  expect(requests.length).toBe(1);
  expect(requests[0].requestHeaders['Content-Type']).not.toBe(
    'application/json;charset=utf-8');
});
```

当没有请求体数据时，我们可以通过遍历头部来把`content-type`请求头删除掉：

```js
function $http(requestConfg) {
  var deferred = $q.defer();
  
  var confg = _.extend({
    method: 'GET'
  }, requestConfg);
  confg.headers = mergeHeaders(requestConfg);

  if (_.isUndefned(confg.data)) {
    _.forEach(confg.headers, function(v, k) {
      if (k.toLowerCase() === 'content-type') {
        delete confg.headers[k];
      }
    });
  }
  // ...
}
```

关于请求头部，我们最后要关注的点是一个请求头部的值不一定是一个字符串，也有可能是一个返回值为字符串的函数。如果`$http`发现传入的头部配置中有函数形式的值，它会调用那个函数，并把函数的返回值作为请求配置对象的头部中的一个属性：

```js
it('supports functions as header values', function() {
  var contentTypeSpy = jasmine.createSpy().and.returnValue(
    'text/plain;charset=utf-8');
  $http.defaults.headers.post['Content-Type'] = contentTypeSpy;
  var request = {
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  };
  $http(request);
  expect(contentTypeSpy).toHaveBeenCalledWith(request);
  expect(requests[0].requestHeaders['Content-Type']).toBe(
    'text/plain;charset=utf-8');
});
```

在`mergeHeaders`的最后，我们会调用一个新的帮助函数`executeHeaderFns`，这个函数会具体负责执行函数形式的头部参数：

```js
function mergeHeaders(confg) {
  // ...
  return executeHeaderFns(reqHeaders, confg);
}
```

这个函数会使用`_.transform`遍历请求头部的对象，如果遇到属性值是函数的情况，会进行调用，并在对应的结果值位置放置：

```js
function executeHeaderFns(headers, confg) {
  return _.transform(headers, function(result, v, k) {
    if (_.isFunction(v)) {
      result[k] = v(confg);
    }
  }, headers);
}
```

还有一种头部生成函数的返回值，不应该在结果值中出现的情况，就是函数返回值为`null`或`undefined`,我们忽略这种返回值：

```js
it('ignores header function value when null/undefned', function() {
  var cacheControlSpy = jasmine.createSpy().and.returnValue(null);
  $http.defaults.headers.post['Cache-Control'] = cacheControlSpy;
  var request = {
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  };
  $http(request);
  expect(cacheControlSpy).toHaveBeenCalledWith(request);
  expect(requests[0].requestHeaders['Cache-Control']).toBeUndefned();
});
```

我们可以在`executeHeaderFns`帮助函数里面对每个头部生成函数进行检测，如果是`null`或者`undefined`，我们可以直接移除这个结果值：

```js
function executeHeaderFns(headers, confg) {
  return _.transform(headers, function(result, v, k) {
    if (_.isFunction(v)) {
      v = v(confg);
      if (_.isNull(v) || _.isUndefned(v)) {
        delete result[k];
      } else {
        result[k] = v;
      }
    }
  }, headers);
}
```
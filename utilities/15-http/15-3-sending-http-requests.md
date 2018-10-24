### 发送 HTTP 请求（Sending HTTP Requests）

加入了 Provider 后，我们就可以开始思考 $http 服务的本职工作了，也就是发送请求到远程服务器并接收响应，这也是本节的目标。

首先，`$http`应该是一个能生成请求的函数，下面是对应的测试用例：

_src/http_spec.js_

```js
'use strict';

var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');

describe('$http', function() {
 
  var $http;
 
  beforeEach(function() {
    publishExternalAPI();
    var injector = createInjector(['ng']);
    $http = injector.get('$http');
  });
  
  it('is a function', function() {
    expect($http instanceof Function).toBe(true);
  });

});
```

$http 函数应该生成一个 HTTP 请求，并返回一个响应对象。但由于 HTTP 请求是异步的，这个函数不会直接返回响应对象，而是返回一个 Promise 对象：

```js
it('returns a Promise', function() {
  var result = $http({});
  expect(result).toBeDefned();
  expect(result.then).toBeDefned();
});
```

我们可以让 $http.$get 直接返回一个函数，这个函数将会创建 Deferred 并返回它的 Promise，这样上面两个用例就可以通过了。为此，我们需要注入 $q 服务：

_src/http.js_

```js
this.$get = ['$httpBackend', '$q', function($httpBackend, $q) {

  return function $http() {
    var deferred = $q.defer();
    return deferred.promise;
  };

}];
```

这就是 $http 服务的最简单方式，接收一个 request 对象，最终返回一个 response 对象，但中间发生了什么呢？我们应该要通过 XMLHttpRequest 创建一个真实的 HTTP 请求，这也是被所有浏览器支持的通信方式。

我们会进行单元测试，但并不想发出真正的网络请求，毕竟特地为此建立一个服务器并不是必要的。这时候，SinonJS 这个我们之前引入的库就能派上用场了。Sinon 可以用伪造的 XMLHttpRequest 对象代替浏览器内置的 XMLHttpRequest 对象。它可以用于检查我们发出了什么请求，并能够直接给出伪造的响应，而不用等待浏览器处理。

要让 Sinon 伪造的 XMLHttpRequest 起作用，我们需要在 beforeEach 函数中提前进行配置。同时，为了让每个用例在启动之前有一个干净的环境，我们也需要 afterEach 钩子函数中把这个伪造请求销魂：

_src/http_spec.js_

```js
'use strict';

var sinon = require('sinon');
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');

describe('$http', function() {
  
  var $http;
  var xhr;

  beforeEach(function() {
    publishExternalAPI();
    var injector = createInjector(['ng']);
    $http = injector.get('$http');
  });

  beforeEach(function() {
    xhr = sinon.useFakeXMLHttpRequest();
  });
  afterEach(function() {
    xhr.restore();
  });

  // ...

});
```

我还需要对发出的请求进行检测。发出请求后，Sinon 会调用伪造 XHR 的`onCreate`回调。如果我们加入了 onCreate 回调，就可以对所有请求进行收集了：

```js
describe('$http', function() {

  var $http;
  var xhr, requests;

  beforeEach(function() {
    publishExternalAPI();
    var injector = createInjector(['ng']);
    $http = injector.get('$http');
  });

  beforeEach(function() {
    xhr = sinon.useFakeXMLHttpRequest();
    requests = [];
    xhr.onCreate = function(req) {
      requests.push(req);
    };
  });
  
  afterEach(function() {
    xhr.restore();
  });
  // ...
});
```

现在我们可以进行第一次的请求测试了。如果我们要利用`$http`"发送一个 POST 请求到 http://teropa.info" 并携带数据'hello'，我现在可以测试这个异步 XHR 是否确实按照要求发送了：

```js
it('makes an XMLHttpRequest to given URL', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 'hello'
  });
  expect(requests.length).toBe(1);
  expect(requests[0].method).toBe('POST');
  expect(requests[0].url).toBe('http://teropa.info');
  expect(requests[0].async).toBe(true);
  expect(requests[0].requestBody).toBe('hello');
});
```

正如之前提到的，`$httpBackend`才是真正承担网络通信任务的，`$http`只是调用它来完成请求，所以 XHR 对象会在`$httpBackend`中被创建。我们先假设`$httpBackend`是一个函数，我们在调用它时传入三个参数，这三个参数都来源于`$http`接收的请求配置对象，分别是请求方法、请求URL和数据：

```js
return function $http(confg) {
  var deferred = $q.defer();
  $httpBackend(confg.method, confg.url, confg.data);
  return deferred.promise;
};
```

_src/http_backend.js_

在`$httpBackend`里面我们会创建一个标准的 XHR 对象，并用传入的参数来进行初始化（open），并发送（send）数据：

```js
this.$get = function() {
  return function(method, url, post) {
    var xhr = new window.XMLHttpRequest();
    xhr.open(method, url, true);
    xhr.send(post || null);
  };
};
```

`xhr.open`使用的三个参数具体是：请求的 HTTP 方法、发送地址 URL 和请求是否是异步的（在 AngularJS 里面所有的请求都是异步的）。

而`xhr.send`的唯一一个参数是要发送的数据。不是所有的请求都带有数据，如果没有带上的话，我们会显式把参数设为`null`。

上面的代码暂时满足了我们测试用例，但还漏了一个重要的步骤：我们在 $http 中返回的 Promise 还一直没有被 resolve，因为我们还没有把它和 XHR 请求关联起来。Promise 应该使用响应对象和对应请求配置对象作为 resolve 的结果值：

```js
it('resolves promise when XHR result received', function() {
  var requestConfg = {
    method: 'GET',
    url: 'http://teropa.info'
  };

  var response;
  $http(requestConfg).then(function(r) {
    response = r;
  });

  requests[0].respond(200, {}, 'Hello');
  
  expect(response).toBeDefned();
  expect(response.status).toBe(200);
  expect(response.statusText).toBe('OK');
  expect(response.data).toBe('Hello');
  expect(response.confg.url).toEqual('http://teropa.info');
});
```

在这个测试中，我们使用 Sinon 的`respond`方法去伪造一个响应，方法参数分别为：HTTP 状态码、HTTP 响应头和响应体。

要使得这种方法行得通，`$httpBackend`不会使用任何 Deferred 和 Promise，而是会接收一个回调。`$httpBackend`中创建的 XHR 会绑定一个`onload`监听函数，一旦`onload`监听函数被调用，回调也会跟着被调用。

```js
this.$get = function() {
  return function(method, url, post, callback) {
    var xhr = new window.XMLHttpRequest();
    xhr.open(method, url, true);
    xhr.send(post || null);
    xhr.onload = function() {
      var response = ('response' in xhr) ? xhr.response :
                                           xhr.responseText;
      var statusText = xhr.statusText || '';
      callback(xhr.status, response, statusText);
    };
  };
};
```

在`onload`监听函数里，我们会首先查看`xhr.response`属性是否包含响应数据，若没有再查看`xhr.responseText`。有些浏览器支持前者，有些支持后者，我们需要兼容处理。另外，我们还会从响应对象中取得响应码和响应描述，并传递到回调中去。

回到`$http`就可以把响应相关的东西都整合起来。我们需要构造回调函数，我们称之为 done。当它被调用时，就会把响应对象作为 resolution，解决掉 Promise：

```js
return function $http(confg) {
  var deferred = $q.defer();

  function done(status, response, statusText) {
    deferred.resolve({
      status: status,
      data: response,
      statusText: statusText,
      confg: confg
    });
  }
  
  $httpBackend(confg.method, confg.url, confg.data, done); 
  return deferred.promise;
};
```

目前的测试用例还没有完全符合我们的期望，问题出在 Promise 被解决的时候，在之前的章节中，回调并不会马上被 resolve，而是会在下一次 digest 中被执行。

我们应该让`$http`在当前没有 digest 运行时，主动调用 digest。我们可以通过`$rootScope`服务的`$apply`方法：

```js
this.$get = ['$httpBackend', '$q', '$rootScope',
  function($httpBackend, $q, $rootScope) {
    
    return function $http(confg) {
      var deferred = $q.defer();

      function done(status, response, statusText) {
        deferred.resolve({
          status: status,
          data: response,
          statusText: statusText,
          confg: confg
        });
        if (!$rootScope.$$phase) {
          $rootScope.$apply();
        }
      }

      $httpBackend(confg.method, confg.url, confg.data, done);
      return deferred.promise;
    };
  }
];
```

这样，我们的测试用例就可以通过了！

这也是 AngularJS 应用中使用`$http`服务的好处之一，框架里会帮我们自动检测调用`$apply`。

第二个好处是当 HTTP 请求出错时，Promise 的错误处理机制就能派上用场。我们可以在请求出错时直接拒绝掉 Promise，而无需一味地 resolve。比如，当网络请求返回`401`：

```js
it('rejects promise when XHR result received with error status', function() {
  var requestConfg = {
    method: 'GET',
    url: 'http://teropa.info'
  };
  
  var response;
  $http(requestConfg).catch(function(r) {
    response = r;
  });

  requests[0].respond(401, {}, 'Fail');

  expect(response).toBeDefned();
  expect(response.status).toBe(401);
  expect(response.statusText).toBe('Unauthorized');
  expect(response.data).toBe('Fail');
  expect(response.confg.url).toEqual('http://teropa.info');
});
```

出错时传递给 Promise 回调的数据也称为响应对象（response object）。区别只在于我们是调用的是 then 还是 catch。

在代码中，我们可以根据响应的状态码决定是对 Deferred 进行 resolve 还是 reject：

```js
function done(status, response, statusText) {
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
```

我们会新建一个`isSuccess`的辅助函数来判断请求是否成功，如果状态码是在 200 到 299 之间的话，就认为请求成功，否则就都判定为请求失败：

```js
function isSuccess(status) {
  return status >= 200 && status < 300;
}
```

> 实际上这样处理会把一些如302的重定向响应也认作请求失败，但由于浏览器内部都会重定向进行处理，所以并不会实际触发 $http 中的错误处理。

请求没有完成也会使得 Promise 被 reject，因为在这种情况下根本不可能会有响应。导致请求不成功有几种原因：可能是网络出错了，也有可能发生跨域资源访问限制，或者请求本身根本已经被禁用。

要想对这种情况进行单元测试，我们可以直接对 XHR 请求对象绑定一个`onerror`回调，这也是 XHR 原生支持的错误处理方式：

```js
it('rejects promise when XHR result errors/aborts', function() {
  var requestConfg = {
    method: 'GET',
    url: 'http://teropa.info'
  };
  
  var response;
  $http(requestConfg).catch(function(r) {
    response = r;
  });

  requests[0].onerror();

  expect(response).toBeDefned();
  expect(response.status).toBe(0);
  expect(response.data).toBe(null);
  expect(response.confg.url).toEqual('http://teropa.info');
});
```

在这种情况下，我们会把响应状态码设置为`0`，把响应数据设置为`null`。

那么在`$httpBackend`中，我们需要为原生 XHR 对象绑定一个`onerror`回调。当它被调用时，我们传递给回调的状态码、响应数据和响应状态描述分别为：-1，null, ''：

```js
return function(method, url, post, callback) {
  var xhr = new window.XMLHttpRequest();
  xhr.open(method, url, true);
  xhr.send(post || null);
  xhr.onload = function() {
    var response = ('response' in xhr) ? xhr.response :
                                         xhr.responseText;
    var statusText = xhr.statusText || '';
    callback(xhr.status, response, statusText);
  };
  xhr.onerror = function() {
    callback(-1, null, '');
  };
};
```

最后要做的是对状态码进行标准化。在错误响应中，`$httpBackend`会返回一个负值作为状态码，而我们在`$http`需要的状态码不可以小于0：

```js
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
```
### 发送 HTTP 请求（Sending HTTP Requests）

加入了 Provider 后，我们就可以开始思考 $http 服务的本职工作了，也就是发送请求到远程服务器并接收响应，这也是本节的目标。

首先，`$http`应该是一个能生成请求的函数，下面是对应的测试用例：

_http_spec.js_

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
## 响应头部（response headers）

现在已经可以通过几种方式设置请求头部，那我们可以把目光转向另一个头部——响应头部。

Angular 支持所有可以从 HTTP 服务器返回的响应头部。它们都将会被放在`$http`服务中的响应对象（response object）之内。响应对象将会有一个 headers 属性指向一个针对头部的访问函数。这个函数就会接收头部名称，并返回对应的头部值：

```js
it('makes response headers available', function() {
  var response;
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  }).then(function(r) {
    response = r;
  });

  requests[0].respond(200, {
    'Content-Type': 'text/plain'
  }, 'Hello');
  
  expect(response.headers).toBeDefned();
  expect(response.headers instanceof Function).toBe(true);
  expect(response.headers('Content-Type')).toBe('text/plain');
  expect(response.headers('content-type')).toBe('text/plain');
});
```

注意就像请求头，我们希望响应头部也不区分大小写，也就是说不管服务器返回的是`Content-Type`还是`content-type`，这两个请求头应该同样生效。

下面，我们开始实现接收 HTTP 服务器的响应对象。当响应回调函数被调用时，需要获取所有从服务器返回的头部，我们可以通过XMLHttpRequest的`getAllResponseHeaders()`方法获取：

_src/http_backend.js_

```js
xhr.onload = function() {
  var response = ('response' in xhr) ? xhr.response :
    xhr.responseText;
  var statusText = xhr.statusText || '';
  callback(
    xhr.status,
    response,
    xhr.getAllResponseHeaders(),
    statusText
  );
};
```

现在，通过`$http`获取到了响应头部（现在依然是一个未被转换为 JS 对象的字符串）后，我们可以定义一个用于头部获取头部的帮助函数——`headersGetter`：

_src/http.js_

```js
function done(status, response, headerString, statusText){
  status = Math.max(status, 0);
  deferred[isSuccess(status) ? 'resolve' : 'reject']({
    status: status,
    data: response,
    statusText: statusText,
    headers: headersGetter(headerString),
    config: config
  })
  if(!$rootScope.$$phase){
    $rootScope.$apply
  }
}
```

headersGetter 帮助函数会接收头部字符串，并返回一个可以根据头部名字获取对应头部值的函数。这个函数也可以被应用开发者使用，用于获取某个请求头部：

```js
function headersGetter(headers) {
  return function(name) {

  };
}
```

要获取特定的响应头，我们需要解析现有的头部字符串。我们会对头部进行懒加载，当我们确实需要获取第一个请求头时才会对请求头部进行解析，并对解析结果进行缓存，以便之后使用：

```js
function headersGetter(headers) {
  var headersObj;
  return function(name) {
    headersObj = headersObj || parseHeaders(headers);
    return headersObj[name.toLowerCase()];
  };
}
```

通过这个模式，我们可以保证在没人需要头部信息的情况下，不会进行头部解析工作，以节省编译成本，提高效率：

真正解析头部的操作会在另一个帮助函数`parseHeaders`中进行,它会对头部字符串进行解析并返回一个 HTTP 头部对象。解析头部对象的第一步就是以换行符为分隔符对字符串进行分离（HTTP 头部通常是每行放一个名值对），然后再对每行进行遍历：


```js
function parseHeaders(headers) {
  var lines = headers.split('\n');
    return _.transform(lines, function(result, line) {

  }, {});
}
```

每行都由一个头部名、一个分号“:”和一个头部值组成。我们只需要获取分号前后的内容，并做一些字符串处理（去除两端的空格、将头部名所有字母变成小写），最后再把处理结果放到结果对象中即可：

```js
function parseHeaders(headers) {
  var lines = headers.split('\n');
  return _.transform(lines, function(result, line) {
    var separatorAt = line.indexOf(':');
    var name = _.trim(line.substr(0, separatorAt)).toLowerCase();
    var value = _.trim(line.substr(separatorAt + 1));
    if (name) {
      result[name] = value;
    }
  }, {});
}
```

headers 函数被调用时也可能不会传入参数，这种情况我们应当返回这种解析好的头部对象：

_test/http_spec.js_

```js
it('may returns all response headers', function() {
  var response;
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: 42
  }).then(function(r) {
    response = r;
  });
  requests[0].respond(200, {
    'Content-Type': 'text/plain'
  }, 'Hello');
  expect(response.headers()).toEqual({
    'content-type': 'text/plain'
  });
});
```

我们可以通过检测是否传入参数来区分：

```js
function headersGetter(headers) {
  var headersObj;
  return function(name) {
    headersObj = headersObj || parseHeaders(headers);
    if (name) {
      return headersObj[name.toLowerCase()];
    } else {
      return headersObj;
    }
  };
}
```

这样，我们就完成了响应头部的部分了！
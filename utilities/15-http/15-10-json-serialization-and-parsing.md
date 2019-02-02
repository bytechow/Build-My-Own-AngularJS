## JSON序列化和解析（JSON Serialization And Parsing）

对于大多 Angular 应用来说，通常都会采用 JSON 作为请求或响应数据的格式。因此，Angular 做了一些处理尽量让关于 JSON 的处理更为简单：如果请求或响应对象是 JSON 格式的，应用开发者将不再需要自行对它们进行转换，Angular 会在底层对其进行自动转换。

对于请求来说，意味着如果你传入的请求数据是一个 JavaScript 对象，它们会在发送前被序列化为 JSON 格式：

_test/http_spec.js_

```js
it('serializes object data to JSON for requests', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: {
      aKey: 42
    }
  });
  
  expect(requests[0].requestBody).toBe('{"aKey":42}');
});
```

如果请求数据是数组，我们也一样会把它进行序列化：

```js
it('serializes array data to JSON for requests', function() {
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: [1, 'two', 3]
  });
  
  expect(requests[0].requestBody).toBe('[1,"two",3]');
});
```

Angular 会在上一节实现的请求数据转换器的基础上实现数据格式化。我们会为`transformRequest`预先设置一个转换器，如果传入的数据是对象（包括数组，因为数组也是对象的一种）的话，这个转换器就会对它进行 JSON 序列化：

_src/http.js_

```js
// var defaults = this.defaults = {
//   headers: {
//     common: {
//       Accept: 'application/json, text/plain, */*'
//     },
//     post: {
//       'Content-Type': 'application/json;charset=utf-8'
//     },
//     put: {
//       'Content-Type': 'application/json;charset=utf-8'
//     },
//     patch: {
//       'Content-Type': 'application/json;charset=utf-8'
//     }
//   },
  transformRequest: [function(data) {
    if (_.isObject(data)) {
      return JSON.stringify(data);
    } else {
      return data;
    }
  }]
// };
```

在这个规则中，还有几个非常重要的例外情况需要考虑。如果响应数据是一个 Blob 类型，其中可能会包含一些二进制或文本数据，我们不应该对这部分的数据进行处理，而只需要原封不动地将它们发送出去就好，XMLHttpRequest 对象会对它们进行处理：

_test/http_spec.js_

```js
it('does not serialize blobs for requests', function() {
  var blob;
  if (window.Blob) {
    blob = new Blob(['hello']);
  } else {
    var BlobBuilder = window.BlobBuilder || window.WebKitBlobBuilder ||
      window.MozBlobBuilder || window.MSBlobBuilder;
    var bb = new BlobBuilder();
    bb.append('hello');
    blob = bb.getBlob('text/plain');
  }
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: blob
  });
  $rootScope.$apply();
  
  expect(requests[0].requestBody).toBe(blob);
});
```

在这个单元测试中，我们需要尝试使用不同的方法来构建 Blob 数据，这是因为各浏览器并没有一个统一的 API 标准，所以需要进行兼容。

我们还需要跳过序列化步骤的是 [FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData)。与 Blob 数据类似，XMLHttpRequest 已经能对 FormData 对象进行处理，我们就无须再将其转换为 JSON 格式：

_test/http_spec.js_

```js
it('does not serialize form data for requests', function() {
  var formData = new FormData();
  formData.append('aField', 'aValue');
  $http({
    method: 'POST',
    url: 'http://teropa.info',
    data: formData
  });
  
  expect(requests[0].requestBody).toBe(formData);
});
```

在我们的转换器中，我们需要先对传入的数据对象进行检测，看它是不是其中一个无须进行序列化的特殊对象。同时，我们也会对第三种特殊对象——`File`进行特殊处理，我们没有专门对它做单元测试，这是因为构建一个文件对象比较麻烦，这并不值得：

> 译者注：反正我们可以确定`File`对象是 XMLHttpRequest 能够进行处理的数据。

_src/http.js_

```js
// transformRequest: [function(data) {
  if (_.isObject(data) && !isBlob(data) &&
    !isFile(data) && !isFormData(data)) {
//     return JSON.stringify(data);
//   } else {
//     return data;
//   }
// }]
```

这里我们需要新增三个辅助函数，每一个函数都通过`toString`这个API来辨识它究竟是不是我们要的那种特殊对象：

```js
function isBlob(object) {
  return object.toString() === '[object Blob]';
}

function isFile(object) {
  return object.toString() === '[object File]';
}

function isFormData(object) {
  return object.toString() === '[object FormData]';
}
```

以上就是关于请求数据 JSON 转换的全部内容。下一个要实现 JSON 转换的是响应数据：如果服务器指定了响应数据的内容类型就是 JSON，那这个响应数据在到达应用代码之前会被转化为一个 JavaScript 数据结构。

_test/http_spec.js_

```js
it('parses JSON data for JSON responses', function() {
  var response;
  $http({
    method: 'GET',
    url: 'http://teropa.info'
  }).then(function(r) {
    response = r;
  });
  requests[0].respond(
    200, {
      'Content-Type': 'application/json'
    },
    '{"message":"hello"}'
  );
  
  expect(_.isObject(response.data)).toBe(true);
  expect(response.data.message).toBe('hello');
});
```

现在我们就需要引入 LoDash 到这个测试文件中来了：

```js
// 'use strict';

var _ = require('lodash');
// var sinon = require('sinon');
// var publishExternalAPI = require('../src/angular_public');
// var createInjector = require('../src/injector');
```

与请求类似，我们会添加一个默认的转换器函数来负责完成转换的工作：

_src/http.js_

```js
// var defaults = this.defaults = {
//   headers: {
//     common: {
//       Accept: 'application/json, text/plain, */*'
//     },
//     post: {
//       'Content-Type': 'application/json;charset=utf-8'
//     },
//     put: {
//       'Content-Type': 'application/json;charset=utf-8'
//     },
//     patch: {
//       'Content-Type': 'application/json;charset=utf-8'
//     }
//   },
//   transformRequest: [function(data) {
//     if (_.isObject(data) && !isBlob(data) &&
//       !isFile(data) && !isFormData(data)) {
//       return JSON.stringify(data);
//     } else {
//       return data;
//     }
//   }],
  transformResponse: [defaultHttpResponseTransform]
// };
```

这个函数跟普通的响应数据转换器一样，会接收响应数据和响应头部作为参数：

```js
function defaultHttpResponseTransform(data, headers) {

}
```

这个函数首先会检查响应数据是否是字符串，还会检查指定的内容类型是否`application/json`。当这两个条件都为`true`，那这个响应数据就会被认为是 JSON 数据并进行解析，而其他的数据类型将不作处理：

```js
function defaultHttpResponseTransform(data, headers) {
  if (_.isString(data)) {
    var contentType = headers('Content-Type');
    if (contentType && contentType.indexOf('application/json') === 0) {
      return JSON.parse(data);
    }
  }
  return data;
}
```

Angular 通常会处理得更聪明一点：如果服务器并没有指明响应内容类型为 JSON，响应数据看起来“像”个JSON，它也会对它进行解析。像下面单元测试的响应数据虽然没指定响应的内容类型，但也会进行解析：

_test/http_spec.js_

```js
it('parses a JSON object response without content type', function() {
  var response;
  $http({
    method: 'GET',
    url: 'http://teropa.info'
  }).then(function(r) {
    response = r;
  });
  requests[0].respond(200, {}, '{"message":"hello"}');

  expect(_.isObject(response.data)).toBe(true);
  expect(response.data.message).toBe('hello');
});
```

这对于数组来说也是一样的：

```js
it('parses a JSON array response without content type', function() {
  var response;
  $http({
    method: 'GET',
    url: 'http://teropa.info'
  }).then(function(r) {
    response = r;
  });
  requests[0].respond(200, {}, '[1, 2, 3]');
  
  expect(_.isArray(response.data)).toBe(true);
  expect(response.data).toEqual([1, 2, 3]);
});
```

因此，在我们的 JSON 响应转换器中，我们就不单单需要检查内容类型参数，也会检查数据本身，看它“像”不“像”一个 JSON 数据？

_src/http.js_

```js
function defaultHttpResponseTransform(data, headers) {
  if (_.isString(data)) {
    var contentType = headers('Content-Type');
    if ((contentType && contentType.indexOf('application/json') === 0) ||
      isJsonLike(data)) {
      return JSON.parse(data);
    }
  }
  return data;
}
```

我们可以简单地判断如果数据字符串中以花括号（{）或方括号（[）开头，就是一个“像”JSON的数据：

```js
function isJsonLike(data) {
  return data.match(/^\{/) || data.match(/^\[/);
}
```
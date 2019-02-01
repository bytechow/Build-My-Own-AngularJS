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
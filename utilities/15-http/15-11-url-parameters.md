## URL 参数（URL Parameters）

到目前为止，我们已经看到`$http`服务发送请求信息的三个部分：请求 URL、请求头部和请求体。接下来我们会讲解最后一种能往 HTTP 请求中加入信息的方式——URL 查询参数（URL _query parameters_）.实际上，也就是在 URL 中问号后面的名值对。

诚然，我们目前的代码已经支持使用查询参数了，毕竟我们可以把查询参数当作 URL 的一部分进行访问。但这样的形式对于参数序列化或跟踪管理上还是比较麻烦，所以可能更希望 Angular 帮你完成这部分的工作。事实上在 Angular 中，我们可以在请求配置中使用`params`属性完成传递 URL 参数的任务：

_test/http_spec.js_

```js
it('adds params to URL', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: 42
    }
  });
  
  expect(requests[0].url).toBe('http://teropa.info?a=42');
});
```

我们还应该对已经含有 query 参数的 URL 进行适配，在这种情况下我们应该把`params`属性传递的参数继续往后拼接上：

```js
it('adds additional params to URL', function() {
  $http({
    url: 'http://teropa.info?a=42',
    params: {
      b: 42
    }
  });
  
  expect(requests[0].url).toBe('http://teropa.info?a=42&b=42');
});
```

在`$http`服务里面，在我们发送请求到 HTTP 后台之前，我们会利用两个辅助函数来完成 URL 的构建，这两个函数分别叫`serializeParams`和`buildUrl`。前者会接受请求参数，并把它们序列化一个 URL 字符串，后者会把请求 URL 和参数字符串拼合在一起：

_src/http.js_

```js
var url = buildUrl(confg.url, serializeParams(confg.params));

// $httpBackend(
//   confg.method,
  url,
//   reqData,
//   done,
//   confg.headers,
//   confg.withCredentials
// );
```

让我们来编写这两个函数。`serializeParams`函数会遍历参数对象，并把里面的每一组参数变成字符串，这个字符串以等号（=）分割参数名和参数值。待遍历完毕，最后再把所有参数组字符串都用`&`连接在一起：

```js
function serializeParams(params) {
  var parts = [];
  _.forEach(params, function(value, key) {
    parts.push(key + '=' + value);
  });
  return parts.join('&');
}
```

`buildUrl`就单纯是将序列化好的参数字符串拼接在原有的 URL 上即可。要注意的是，它会根据传入的 URL 是否包含`?`，来决定是用`?`还是`&`作为连接的字符：

```js
function buildUrl(url, serializedParams) {
  if (serializedParams.length) {
    url += (url.indexOf('?') === -1) ? '?' : '&';
    url += serializedParams;
  }
  return url;
}
```

URL 参数可能会含有一些特殊字符，无法直接拼接到 URL 上。比如`=`和`&`，显然很容易会与参数分隔符产生混淆。因此，我们需要先对特殊字符进行转义：

_test/http_spec.js_

```js
it('escapes url characters in params', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      '==': '&&'
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?%3D%3D=%26%26');
});
```

JavaScript 原生支持的`encodeURIComponent`就可以用于转义：

_src/http.js_

```js
// function serializeParams(params) {
//   var parts = [];
//   _.forEach(params, function(value, key) {
//     parts.push(
      encodeURIComponent(key) + '=' + encodeURIComponent(value));
//   });
//   return parts.join('&');
// }
```

> 实际上，AngularJS 并不会直接使用`encodeURIComponent`来进行转码，而是采用内建的工具方法——`encodeUriQuery`。这个方法的转义范围并不会像`encodeURIComponent`那样广，它不会对`@`和`:`进行转义。

如果传入请求配置对象的参数值是`null`和`undefined`，这个参数就会被忽略掉：

_test/http_spec.js_

```js
it('does not attach null or undefned params', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: null,
      b: undefned
    }
  });
  expect(requests[0].url).toBe('http://teropa.info');
});
```

我们可以在便利时加入一个检测，如果值为空则跳过：

```js
function serializeParams(params) {
  var parts = [];
  _.forEach(params, function(value, key) {
    if (_.isNull(value) || _.isUndefned(value)) {
      return;
    }
    parts.push(
      encodeURIComponent(key) + '=' + encodeURIComponent(value));
  });
  return parts.join('&');
}
```

HTTP 协议支持使用同一个参数名传递多个值。这块可以通过重用同一个参数名来完成，Angular 也支持这种传参方式，我们会使用数组的形式来存储这类同名参数值：

_test/http_spec.js_

```js
it('attaches multiple params from arrays', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: [42, 43]
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?a=42&a=43');
});
```

为此，我们需要在每次参数遍历时，在内部也进行一次内部的遍历。为了兼容所有类型的参数，我们把所有非数组类型的参数都包裹在数组里：

_src/http.js_

```js
// function serializeParams(params) {
//   var parts = [];
//   _.forEach(params, function(value, key) {
//     if (_.isNull(value) || _.isUndefned(value)) {
//       return;
//     }
    if (!_.isArray(value)) {
      value = [value];
    }
    _.forEach(value, function(v) {
      parts.push(
        encodeURIComponent(key) + '=' + encodeURIComponent(v));
    });
//   });
//   return parts.join('&');
// }
```

因此，数组用作参数值时是有特殊用途的。那对象呢？HTTP 协议默认并不支持“嵌套参数”（nested parameter）,那如果我们传入对象作为参数值会发生什么呢（之后需要进行 URL 转义）。事实上，如果服务器已经准备好对 JSON 数据进行解析，我们也可以在 query 参数中传递：

_test/http_spec.js_

```js
it('serializes objects to json', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: { b: 42 }
    }
  });

  expect(requests[0].url).toBe('http://teropa.info?a=%7B%22b%22%3A42%7D');
});
```

在我们对参数值进行遍历时，我们需要检测这个值是不是对象类型，若是需要将其转成 JSON 数据：

_src/http.js_

```js
// function serializeParams(params) {
//   var parts = [];
//   _.forEach(params, function(value, key) {
//     if (_.isNull(value) || _.isUndefned(value)) {
//       return;
//     }
//     if (!_.isArray(value)) {
//       value = [value];
//     }
//     _.forEach(value, function(v) {
      if (_.isObject(v)) {
        v = JSON.stringify(v);
      }
//       parts.push(
//         url += encodeURIComponent(key) + '=' + encodeURIComponent(v));
//     });
//   });
//   return parts.join('&');
// }
```

要做 JSON 序列化，就需要考虑对日期的处理，在序列化（JSON.stringify）的过程中，Date 类型的值会自动转变为 ISO 8601 格式。为了验证这个事实，我们会多增加一个单元测试：

_test/http_spec.js_

```js
it(‘serializes dates to ISO strings’, function() {
  $http({
    url: ‘http: //teropa.info’,
      params: {
        a: new Date(2015, 0, 1, 12, 0, 0)
      }
  });
  $rootScope.$apply();
  
  expect(/\d{4}-\d{2}-\d{2}T\d{2}%3A\d{2}%3A\d{2}/
    .test(requests[0].url)).toBeTruthy();
});
```

确实，我们对日期类型的 URL 参数进行序列化，默认的结果就是会输出为 ISO 8601 格式。而对于 Angular 应用开发者来说，你也可以使用其他序列化方法来代替。你可以在请求配置对象或`$http`默认配置中添加一个`paramSerializer`属性，它是一个数组，可以把传入的对象参数转化为字符串。它的作用本质上与我们之前实现的`serializeParams`一致：

```js
it('allows substituting param serializer', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: 42,
      b: 43
    },
    paramSerializer: function(params) {
      return _.map(params, function(v, k) {
        return k + '=' + v + 'lol';
      }).join('&');
    }
  });
  
  expect(requests[0].url)
    .toEqual('http://teropa.info?a=42lol&b=43lol');
});
```

因此，当我们构建请求 URL 时，是会调用请求配置中的`paramSerializer`函数，而不是直接调用`serializeParams`函数：

_src/http.js_

```js
var url = buildUrl(confg.url,
  confg.paramSerializer(confg.params));

<!-- $httpBackend(
  confg.method,
  url,
  reqData,
  done,
  confg.headers,
  confg.withCredentials
); -->
```

而在请求的全局默认配置中，我们会把`serializeParams`作为`paramSerializer`的默认值。没有主动传入自定义的序列化方法是，我们默认还是使用`serializeParams`：

```js
var defaults = this.defaults = {
  // ...
  paramSerializer: serializeParams
};
```

在构建实际的请求配置对象时，我们会默认加入全局默认配置中的序列化方法，也就是`serializeParams`：

```js
function $http(requestConfg) {
  var confg = _.extend({
    method: 'GET',
    transformRequest: defaults.transformRequest,
    transformResponse: defaults.transformResponse,
    paramSerializer: defaults.paramSerializer
  }, requestConfg);
  // ...
}
```

实际上，还有一个更便利的方法可以设置我们自定义的序列化方法：我们可以注册一个 Angular 服务，使得它可以被依赖注入到请求配置中去，当注册好了后，我们只需要传入服务的名称即可：

_test/http_spec.js_

```js
it('allows substituting param serializer through DI', function() {
  var injector = createInjector(['ng', function($provide) {
    $provide.factory('mySpecialSerializer', function() {
      return function(params) {
        return _.map(params, function(v, k) {
          return k + '=' + v + 'lol';
        }).join('&');
      };
    });
  }]);
  injector.invoke(function($http) {
    $http({
      url: 'http://teropa.info',
      params: {
        a: 42,
        b: 43
      },
      paramSerializer: 'mySpecialSerializer'
    });
    expect(requests[0].url)
      .toEqual('http://teropa.info?a=42lol&b=43lol');
  });
});
```

要使用依赖注入，也就必须先引入`$inject`服务：

_src/http.js_

```js
this.$get = ['$httpBackend', '$q', '$rootScope', '$injector',
              function($httpBackend, $q, $rootScope, $injector) {
  // ...
}];
```

在`$http`函数中，如果我们检测到参数序列化的参数值为一个字符串，我们就可以使用`$inject`服务来获取对应的参数序列化方法：

```js
// function $http(requestConfg) {
//   var confg = _.extend({
//     method: 'GET',
//     transformRequest: defaults.transformRequest,
//     transformResponse: defaults.transformResponse,
//     paramSerializer: defaults.paramSerializer
//   }, requestConfg);
//   confg.headers = mergeHeaders(requestConfg);
  if (_.isString(confg.paramSerializer)) {
    confg.paramSerializer = $injector.get(confg.paramSerializer);
  }
  // ...
// }
```

事实上，默认的参数序列化方法本身就是一个可供依赖注入的服务，名为`$httpParamSerializer`。这就意味着你可以注入它用作他途或者对它进行装饰（decorate）：

_src/angular_public_spec.js_

```js
it('makes default param serializer available through DI', function() {
  var injector = createInjector(['ng']);
  injector.invoke(function($httpParamSerializer) {
    var result = $httpParamSerializer({
      a: 42,
      b: 43
    });
    expect(result).toEqual('a=42&b=43');
  });
});
```

在`http.js`中，我们要对`serializeParams`函数进行改变，让它不再是顶层的函数，而是一个 provider 服务的返回值：

_src/http.js_

```js
function $HttpParamSerializerProvider() {
  this.$get = function() {
    return function serializeParams(params) {
      // var parts = [];
      // _.forEach(params, function(value, key) {
      //   if (_.isNull(value) || _.isUndefned(value)) {
      //     return;
      //   }
      //   if (!_.isArray(value)) {
      //     value = [value];
      //   }
      //   _.forEach(value, function(v) {
      //     if (_.isObject(v)) {
      //       v = JSON.stringify(v);
      //     }
      //     parts.push(
      //       encodeURIComponent(key) + '=' + encodeURIComponent(v));
      //   });
      // });
      // return parts.join('&');
    };
  };
}
```

我们还要调整`http.js`文件的输出模块（export），把上面我们加入的参数序列化服务进行对外输出：

```js
module.exports = {
  $HttpProvider: $HttpProvider,
  $HttpParamSerializerProvider: $HttpParamSerializerProvider
};
```

接下来，我们会在`ng`模块中注册这个服务：

_src/angular_public.js_

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  
  var ngModule = angular.module('ng', []);
  ngModule.provider('$flter', require('./flter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
  ngModule.provider('$q', require('./q').$QProvider);
  ngModule.provider('$$q', require('./q').$$QProvider);
  ngModule.provider('$httpBackend', require('./http_backend'));
  ngModule.provider('$http', require('./http').$HttpProvider);
  ngModule.provider('$httpParamSerializer',
    require('./http').$HttpParamSerializerProvider);
}
```

在`$http`默认全局配置中，参数序列化的默认值就要变成`$httpParamSerializer`了，因为已经没有独立的`serializeParams`函数了：

_src/http.js_

```js
var defaults = this.defaults = {
  // ...
  paramSerializer: '$httpParamSerializer'
};
```

Angular 默认还提供了一个`$httpParamSerializer`的替代服务——`$httpParamSerializerJQLike`服务，这个服务可以使用 jQuery 的序列化方法。这个方法既适用于与一些使用 jQuery 序列化格式的后台进行通信，或者在无法使用时 JSON 时传送嵌套数据。

之后我们会看到，`$httpParamSerializer`会对集合（collections）进行特殊处理，但对于原始类型来说，该服务与默认服务的处理一致：

_test/http_spec.js_

```js
describe('JQ-like param serialization', function() {

  it('is possible', function() {
    $http({
      url: 'http://teropa.info',
      params: {
        a: 42,
        b: 43
      },
      paramSerializer: '$httpParamSerializerJQLike'
    });

    expect(requests[0].url).toEqual('http://teropa.info?a=42&b=43');
  });
});
```

这个服务将会在`http.js`里被单独放在一个 provider 里：

_src/http.js_

```js
function $HttpParamSerializerJQLikeProvider() {
  this.$get = function() {
    return function(params) {
      var parts = [];
      _.forEach(params, function(value, key) {
        parts.push(
          encodeURIComponent(key) + '=' + encodeURIComponent(value));
      });
      return parts.join('&');
    };
  };
}
```

接着，我们也要对外提供这个 provider：

```js
module.exports = {
  $HttpProvider: $HttpProvider,
  $HttpParamSerializerProvider: $HttpParamSerializerProvider,
  $HttpParamSerializerJQLikeProvider: $HttpParamSerializerJQLikeProvider
};
```

并需要在`ng`模块注册：

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  var ngModule = angular.module('ng', []);
  ngModule.provider('$flter', require('./flter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
  ngModule.provider('$q', require('./q').$QProvider);
  ngModule.provider('$$q', require('./q').$$QProvider);
  ngModule.provider('$httpBackend', require('./http_backend'));
  ngModule.provider('$http', require('./http').$HttpProvider);
  ngModule.provider('$httpParamSerializer',
    require('./http').$HttpParamSerializerProvider);
  ngModule.provider('$httpParamSerializerJQLike',
    require('./http').$HttpParamSerializerJQLikeProvider);
}
```

这个服务会跳过`null`和`undefined`这类值：

```js
function $HttpParamSerializerJQLikeProvider() {
  this.$get = function() {
    return function(params) {
      var parts = [];
      _.forEach(params, function(value, key) {
        if (_.isNull(value) || _.isUndefned(value)) {
          return;
        }
        parts.push(
          encodeURIComponent(key) + '=' + encodeURIComponent(value));
      });
      return parts.join('&');
    };
  };
}
```

这个服务与默认的序列化服务的区别出现在处理数组上。如果我们需要传递数组类型的参数（同名参数），我们会在参数名的后面加入双方括号`[]`后缀：

```js
it('uses square brackets in arrays', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: [42, 43]
    },
    paramSerializer: '$httpParamSerializerJQLike'
  });
  
  expect(requests[0].url).toEqual('http://teropa.info?a%5B%5D=42&a%5B%5D=43');
});
```

上面可以看出，开方括号`[`对应的 URL 编码是`%5B`，而`]`对应的 URL 编码是`%5D`。

下面我们就看看怎么处理数组：

_src/http.js_

```js
// function $HttpParamSerializerJQLikeProvider() {
//   this.$get = function() {
//     return function(params) {
//       var parts = [];
//       _.forEach(params, function(value, key) {
//         if (_.isNull(value) || _.isUndefned(value)) {
//           return;
//         }
        if (_.isArray(value)) {
          _.forEach(value, function(v) {
            parts.push(
              encodeURIComponent(key + '[]') + '=' + encodeURIComponent(v));
          });
        } else {
          // parts.push(
          //   encodeURIComponent(key) + '=' + encodeURIComponent(value));
        }
//       });
//       return parts.join('&');
//     };
//   };
// }
```

而在序列化对象的时候，我们也会用到方括号。与数组不同的是，我们会用方括号来包裹对象的属性名（key）：

_test/http_spec.js_

```js
it('uses square brackets in objects', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: {
        b: 42,
        c: 43
      }
    },
    paramSerializer: '$httpParamSerializerJQLike'
  });
  
  expect(requests[0].url).toEqual('http://teropa.info?a%5Bb%5D=42&a%5Bc%5D=43');
});
```

我们会在`else if`分支中对对象进行处理。要注意的是，Date 类型也算是对象，但我们应该把它们作为原始类型来进行序列化。于是，我们要加上一个额外的判断：

_src/http.js_

```js
// function $HttpParamSerializerJQLikeProvider() {
//   this.$get = function() {
//     return function(params) {
//       var parts = [];
//       _.forEach(params, function(value, key) {
//         if (_.isNull(value) || _.isUndefned(value)) {
//           return;
//         }
//         if (_.isArray(value)) {
//           _.forEach(value, function(v) {
//             parts.push(
//               encodeURIComponent(key + '[]') + '=' + encodeURIComponent(v));
//           });
//         } 
        else if (_.isObject(value) && !_.isDate(value)) {
          _.forEach(value, function(v, k) {
            parts.push(
              encodeURIComponent(key + '[' + k + ']') + '=' +
              encodeURIComponent(v));
          });
//         } else {
//           parts.push(
//             encodeURIComponent(key) + '=' + encodeURIComponent(value));
//         }
//       });
//       return parts.join('&');
//     };
//   };
// }
```

方括号前缀还支持递归，所以我们就可以用这种方式来表示嵌套对象，如`{a: {b: {c: 42}}}`会变成`a[b][c]=42`：

_test/http_spec.js_

```js
it('supports nesting in objects', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: {
        b: {
          c: 42
        }
      }
    },
    paramSerializer: '$httpParamSerializerJQLike'
  });

  expect(requests[0].url).toEqual('http://teropa.info?a%5Bb%5D%5Bc%5D=42');
});
```

我们需要对代码进行递归化的改造，以支持任何深度的嵌套。我们要对代码进行重新组织，首先要构建一个内部函数`serialize`，这个函数始终会在当前遍历到的属性名（key）前面加上方括号前缀，如果发现还有嵌套的结构，它还没会对自身进行递归调用。直到递归遍历发现叶子节点（既不是对象，也不是数组）层次的值之前，我们都不会把结果加入到`parts`这个结果数组中：

_src/http.js_

```js
// function $HttpParamSerializerJQLikeProvider() {
//   this.$get = function() {
//     return function(params) {
//       var parts = [];

      function serialize(value, prefx) {
        if (_.isNull(value) || _.isUndefned(value)) {
          return;
        }
        if (_.isArray(value)) {
          _.forEach(value, function(v) {
            serialize(v, prefx + '[]');
          });
        } else if (_.isObject(value) && !_.isDate(value)) {
          _.forEach(value, function(v, k) {
            serialize(v, prefx + '[' + k + ']');
          });
        } else {
          parts.push(
            encodeURIComponent(prefx) + '=' + encodeURIComponent(value));
        }
      }
      
      // _.forEach(params, function(value, key) {
      //   if (_.isNull(value) || _.isUndefned(value)) {
      //     return;
      //   }
      //   if (_.isArray(value)) {
      //     _.forEach(value, function(v) {
            serialize(v, key + '[]');
        //   });
        // } else if (_.isObject(value) && !_.isDate(value)) {
        //   _.forEach(value, function(v, k) {
            serialize(v, key + '[' + k + ']');
//           });
//         } else {
//           parts.push(
//             encodeURIComponent(key) + '=' +
//             encodeURIComponent(value));
//         }
//       });
//       return parts.join('&');
//     };
//   };
// }
```

目前的代码已经能很好地实现了我们的需求了，但可以发现我们还是重复了一部分的代码。我们在递归的顶层写了一次，又在递归时写了这段逻辑，其实两者的唯一区别就在于递归顶层不需要使用方括号：

```js
// function $HttpParamSerializerJQLikeProvider() {
//   this.$get = function() {
//     return function(params) {
//       var parts = [];

      function serialize(value, prefx, topLevel) {
        // if (_.isNull(value) || _.isUndefned(value)) {
        //   return;
        // }
        // if (_.isArray(value)) {
        //   _.forEach(value, function(v) {
        //     serialize(v, prefx + '[]');
        //   });
        // } else if (_.isObject(value) && !_.isDate(value)) {
        //   _.forEach(value, function(v, k) {
            serialize(v, prefx +
              (topLevel ? '' : '[') +
              k +
              (topLevel ? '' : ']'));
      //     });
      //   } else {
      //     parts.push(
      //       encodeURIComponent(prefx) + '=' + encodeURIComponent(value));
      //   }
      // }

      serialize(params, '', true);
      
//       return parts.join('&');
//     };
//   };
// }
```

最后，我们要考虑一种情况，就是数组中包含对象。这时我们会用数组元素的索引代替属性名，比如`{a: [{b: 42}]}`不会变成`a[][b]=42`，而是`a[0][b]=42`：

_test/http_spec.js_

```js
it('appends array indexes when items are objects', function() {
  $http({
    url: 'http://teropa.info',
    params: {
      a: [{
        b: 42
      }]
    },
    paramSerializer: '$httpParamSerializerJQLike'
  });
  expect(requests[0].url).toEqual('http://teropa.info?a%5B0%5D%5Bb%5D=42');
});
```

我们可以直接利用遍历方法中的遍历索引值替换即可：

_src/http.js_

```js
// function $HttpParamSerializerJQLikeProvider() {
//   this.$get = function() {
//     return function(params) {
//       var parts = [];

//       function serialize(value, prefx, topLevel) {
//         if (_.isNull(value) || _.isUndefned(value)) {
//           return;
//         }
//         if (_.isArray(value)) {
          _.forEach(value, function(v, i) {
            // serialize(v, prefx +
            //   '[' +
              (_.isObject(v) ? i : '') +
//               ']');
//           });
//         } else if (_.isObject(value)) {
//           _.forEach(value, function(v, k) {
//             serialize(v, prefx +
//               (topLevel ? '' : '[') +
//               k +
//               (topLevel ? '' : ']'));
//           });
//         } else {
//           parts.push(
//             encodeURIComponent(prefx) + '=' + encodeURIComponent(value));
//         }
//       }
//       serialize(params, '', true);
//       return parts.join('&');
//     };
//   };
// }
```
### 处理循环依赖问题（Circular Dependencies）

既然支持依赖注入其它依赖，我们就可能会遇到循环依赖问题。假设A依赖B，B依赖C，C又依赖A，一旦我们构建依赖就会陷入死循环，最终抛出堆栈溢出的错误。但是我们希望有一个更清晰的错误提示：

test/injector\_spec.js

```
it('notifes the user about a circular dependency', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', {
    $get: function(b) {}
  });
  module.provider('b', {
    $get: function(c) {}
  });
  module.provider('c', {
    $get: function(a) {}
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrowError(/Circular dependency found/);
});
```

要满足这一需求，有两个关键点。第一个关键点在于，当我们调用某个 provider 的 $get 方法生成依赖之前时，我们会在其对应的实例依赖中加入一个标识（marker），这个标识可以记录该 provider 依赖的状态为“正在生成中”，这样当下次我们再次访问这个依赖时，我们就知道发生了循环依赖。

我们可以采用一个空对象作为标识（marker）,因为空对象不等于除它自己以外的任何值：

src/injector.js

```js
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /(\/\/.*$)|(\/\*.*?\*\/)/mg;
var INSTANTIATING = { };
```

在 getService 函数中，我们会在调用 $provider 之前在对应的实例依赖放入标识，并在每次调用 getService 时先检查对应的实例依赖是否已经被标记，若已经被标记，则说明出现了循环依赖：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    if (instanceCache[name] === INSTANTIATING) {
      throw new Error('Circular dependency found');
    }
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    instanceCache[name] = INSTANTIATING;
    var provider = providerCache[name + 'Provider'];
    var instance = instanceCache[name] = invoke(provider.$get);
    return instance;
  }
}
```

然后当




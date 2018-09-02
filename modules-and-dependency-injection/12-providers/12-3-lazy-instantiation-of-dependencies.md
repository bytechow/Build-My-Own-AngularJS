### 依赖的懒加载（Lazy Instantiation of Dependencies）

有依赖包含依赖的情况存在，我们就需要考虑依赖加载顺序的问题。我们创建一个依赖时，该依赖所包含的依赖是否都已经可用了？思考下面的用例，当组件 b 依赖组件 a ，但 a 在 b 之后注册：

test/injector\_spec.js

```js
it('injects the $get method of a provider lazily', function() {
  var module = window.angular.module('myModule', []);
  module.provider('b', {
    $get: function(a) {
      return a + 2;
    }
  });
  module.provider('a', {
    $get: _.constant(1)
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('b')).toBe(3);
});
```

测试用例没有通过，这是因为组件 a 在被注入时还未就绪。

如果 Angular 的注射器真的是这么工作的话，我们必须时刻注意依赖加载的顺序。幸亏注射器实际上采用的是懒加载的模式——只有当 $get 方法的返回值（第一次）被需要，才会调用 $get 进行实例化。所以，在本例中，组件 b 虽然注册在先，但它的 $get 方法并不会立即执行。只有当我们创建一个依赖组件 b 的注射器时，$get 方法才会被真正执行，这时 a 肯定已经注册好了。

由于我们需要 provider 的 $get 方法延后调用，所以我们需要有一个空间存储 provider 对象。我们自然而然地想到了之前使用的 cache 对象，但 cache 对象存储的应该只能是依赖实例，而不是 provider。所以我们要使用两个存储空间进行保存，一个用于存放 provider，另一个存放依赖实例：

src/injector.js

```js
function createInjector(modulesToLoad, strictDi) {
  var providerCache = {};
  var instanceCache = {};
  var loadedModules = {};
  // ...
}
```

然后在 $provider 对象中，我们将 constant 放在 instanceCache，将 provider 放到 providerCache。当接收到一个 provider，我们以 provider 名称再加上后缀 “Provider”作为 key 进行存储，这样我们就能从命名上起到区分的作用：

src/injector.js

```js
var $provide = {
  constant: function(key, value) {
    if (key === 'hasOwnProperty') {
      throw 'hasOwnProperty is not a valid constant name!';
    }
    instanceCache[key] = value;
  },
  provider: function(key, provider) {
    providerCache[key + 'Provider'] = provider;
  }
};
```

当真正需要 provider 生成依赖值时（要么被依赖注入，要么是直接访问），我们需要检索这两个缓存。让我们在 createInjector 里新建一个内部方法 getService 完成这项工作。如果调用函数时传入一个依赖名称，我们会先检索依赖实例缓存（instance cache），若找到则直接返回，若没有找到，再检索 provider 缓存。如果检索到的是一个 provider，那么就会像之前那样使用 invoke 方法进行调用：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    var provider = providerCache[name + 'Provider'];
    return invoke(provider.$get, provider);
  }
}
```

现在，在 invoke 方法里面我们也要使用 getService 方法代替直接的对象属性查找了：

src/injector.js

```js
function invoke(fn, self, locals) {
  var args = _.map(annotate(fn), function(token) {
    if (_.isString(token)) {
      return locals && locals.hasOwnProperty(token) ?
        locals[token] :
        getService(token);
    } else {
      throw 'Incorrect injection token! Expected a string, got ' + token;
    }
  });
  if (_.isArray(fn)) {
    fn = _.last(fn);
  }
  return fn.apply(self, args);
}
```

同样，其他依赖属性检索的接口也要跟着变动，比如 has 和 get 方法：

src/injector.js

```js
return {
  has: function(key) {
    return instanceCache.hasOwnProperty(key) ||
      providerCache.hasOwnProperty(key + 'Provider');
  },
  get: getService,
  annotate: annotate,
  invoke: invoke
};
```

从上面的代码实现，我们知道了只有当 provider 依赖被注入过或者被显式调用过，provider 依赖才会被实例化。如果没有任何地方对依赖进行获取，这个依赖的 $get 方法将不会被执行，也就不会生成依赖了。

你可以通过 injector.has 方法检查依赖是否已注册，但不能保证这个依赖已经被实例化了，毕竟有可能找到的是一个 provider。


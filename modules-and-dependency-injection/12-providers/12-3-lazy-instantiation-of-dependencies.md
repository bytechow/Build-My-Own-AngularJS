### 依赖的懒实例化（Lazy Instantiation of Dependencies）

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

如果 Angular 的注射器真的是这么工作的话，我们必须时刻注意依赖加载的顺序。幸亏注射器实际上采用的是懒加载的模式。只有当 $get 方法的返回值（第一次）被需要，才会进行实例化。所以，在本例中，组件 b 虽然注册在先，但它的 $get 方法并不会立即执行。只有当我们创建一个依赖组件 b 的注射器时，$get 方法才会被真正执行，这时 a 肯定已经注册好了。

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




### 注入 $injector（Injecting The $injectors）

当你创建了一个 injector 对象，你就可以使用它的 API 来引入、注入依赖。对于应用开发者来说，可能其中最有趣的 API 是 get 方法，因为使用它可以动态获取一个依赖，甚至不用理会它的名称是否已放到了注解中进行声明。

让应用开发者更便利地获取 injector 实例本身，是非常有价值的。Angular也是这样做的，injector 实例可以通过 $inject 这个依赖名称进行获取：

test/injector_spec.js

```js
it('allows injecting the instance injector to $get', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 42);
  module.provider('b', function BProvider() {
    this.$get = function($injector) {
      return $injector.get('a');
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('b')).toBe(42);
});
```

这里我们注入的 $injector 实际上就是 createInjector 生成出来并作为返回值的实例注射器（instance injector）。我们可以直接对实例注射器新增属性 $injector，赋值为实例注射器实例：

src/injector.js

```js
var instanceInjector = instanceCache.$injector =
  createInternalInjector(instanceCache, function(name) {
    var provider = providerInjector.get(name + 'Provider');
    return instanceInjector.invoke(provider.$get, provider);
  });
```

当然，你也可以在 provider 构造函数中注入 $injector，而 $injector 承载的是 provider 注射器实例，所以根据我们之前的实现规则，$injector 只能访问到其他 provider 和常量:

test/injector_spec.js

```js
it('allows injecting the provider injector to provider', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.value = 42;
    this.$get = function() {
      return this.value;
    };
  });
  module.provider('b', function BProvider($injector) {
    var aProvider = $injector.get('aProvider');
    this.$get = function() {
      return aProvider.value;
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('b')).toBe(42);
});
```

同理，我们可以实现 provider 的 $injector 依赖：

src/injector.js

```js
var providerInjector = providerCache.$injector =
  createInternalInjector(providerCache, function() {
    throw 'Unknown provider: '+path.join(' <- ');
  });
```

所以，我们在注入 $injector 时可能获取到的是 instance injector，也可能是 provider injector，这取决于我们在哪里注入它。

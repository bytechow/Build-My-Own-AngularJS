### 注入 $provider（Injecting $provider）

injector 实例对象可以让你获取已经配置好的依赖，但你没办法对依赖进行配置，也就是说这是一个只读的API。如果你希望配置依赖，你需要使用 $provder 依赖。

通过注入的 $provider，我们可以调用之前任务队列的生成方法来进行一些前置的配置。比如，我们可以在 provider 构造函数中注册一个常量：

test/injector_spec.js

```js
it('allows injecting the $provide service to providers', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider($provide) {
    $provide.constant('b', 2);
    this.$get = function(b) {
      return 1 + b;
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(3);
});
```

要注意的是，$provider 只是在 provider 构造函数中可用，运行时中不能再使用 $provder 进行配置或者增加依赖项：

test.injector_spec.js

```js
it('does not allow injecting the $provide service to $get', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function($provide) {};
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrow();
});
```

实际上，我们要注入的 $provider 就是我们在 createInjector 中声明的内部函数 $provider，它目前拥有 constant 和 provider 方法。所以，我们现在知道为什么叫这个名字了


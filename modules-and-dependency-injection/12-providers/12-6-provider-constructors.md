### Provider 构造器（Provider Constructors）

之前我们把 provider 定义一个包含 $get 方法的对象。这种说法没什么问题，但是我们也可能会使用一个构造函数来生成一个 provider：

```js
function AProvider() {
  this.$get = function() { return 42; }
}
```

上例的构造函数调用之后会生成一个带有 $get 方法的对象，也就是我们之前定义的 provider，但还有另一种方式：

```js
function AProvider() {
  this.value = 42;
}
AProvider.prototype.$get = function() {
  return this.value;
}
```

所以，只要构造函数能够在调用后返回一个能访问 $get 方法的对象，都算是 provider 构造器。你甚至可以使用 ES2015 中的 class 语法糖和继承机制。

下面是对应的测试用例：

test/injector\_spec.js

```js
it('instantiates a provider if given as a constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function() {
      return 42;
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

另外，我们还允许 provider 构造函数注入其它依赖：

test/inject\_spec.js

```js
it('injects the given provider constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.constant('b', 2);
  module.provider('a', function AProvider(b) {
    this.$get = function() {
      return 1 + b;
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(3);
});
```

要实现构造函数形式的 provider，我们需要在组件注册时进行检验判断。如果 provider 函数是一个函数，那我们需要进行实例化，可以使用我们在前面章节实现的 instantiate：

src/injector.js

```js
provider: function(key, provider) {
  if (_.isFunction(provider)) {
    provider = instantiate(provider);
  }
  providerCache[key + 'Provider'] = provider;
}
```

现在我们有两种为 provider 注入依赖的方式：使用构造函数的参数，或者使用 provider.$get 方法。当然，我们也可以从代码实现中发现，构造函数方式的注入没有办法做到懒加载。当使用 Provider 构造函数进行 provider 的注册，其构造函数就会马上被调用，如果在此之前该 provider 的某个依赖还没就绪，就会出错。

实际上，这两种 provider 并不是可以互换的，它们各有应用场景。


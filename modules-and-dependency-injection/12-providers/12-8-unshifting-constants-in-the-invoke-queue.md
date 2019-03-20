### 提升常量在任务队列的注册优先级 （Unshifting Constants in The Invoke Queue）

正如我们之前实现的，实例的懒加载能让应用开发者不用严格按照依赖顺序注册依赖。即使 B 依赖 A，A 也可以在 B 之后被注册。

但 provider 构造函数就没有这么自由了。provider 构造函数会在注册的时候就被加载（也就是在执行任务队列时就会进行实例化）。如果 BProvider 依赖 AProvider，那么 AProvider 就必须在 BProvider 之前注册，在这种情况下，Angular 并不会帮你重新调整顺序。

但对于常量来说，Angular 却会对它进行重新排序。具体来说，就是会把注册常量的任务提到前面去。这样一来，即使在 provider 构造函数中，也可以注入后面注册的常量。

test/injector\_spec.js

```js
it('registers constants first to make them available to providers', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider(b) {
    this.$get = function() {
      return b;
    };
  });
  module.constant('b', 42);
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

当模块中注册了常量，模块加载器都会把它们放到任务队列的最前面，因此就可以无视依赖顺序。当然，因为常量并不依赖其他东西，所以我们这么做才是安全的。

我们可以在 invokeLater 内部函数中加入一个可选的参数，这个参数可以指定注册依赖的任务会以何种形式插入到任务队列中，默认是 push 方法，也就是会放到队列的最后：

```js
var invokeLater = function(method, arrayMethod){
  return function(){
    invokeQueue[arrayMethod || 'push']([method, arguments]);
    return moduleInstance;
  };
};
```

对于注册常量的任务，我们会采用 unshift 方法，也就是在队列的开头插入：

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('constant', 'unshift'),
  provider: invokeLater('provider'),
  _invokeQueue: invokeQueue
};
```




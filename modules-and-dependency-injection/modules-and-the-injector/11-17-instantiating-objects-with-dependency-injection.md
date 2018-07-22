### 使用依赖注入实例化对象（Instantiating Objects with Dependency Injection）

我们将要实现本章最后一个特性：使用构造函数进行依赖注入。

为了实现这一个功能，我们将要实现 injector.instantiate 方法。首先，我们要支持声明式依赖注入：

test/injector_spec.js

```js
it('instantiates an annotated constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  function Type(one, two) {
    this.result = one + two;
  }
  Type.$inject = ['a', 'b'];
  var instance = injector.instantiate(Type);
  expect(instance.result).toBe(3);
}); 
```

也要支持行内式依赖注入：

test/injector_spec.js

```js
it('instantiates an array-annotated constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  function Type(one, two) {
    this.result = one + two;
  }
  var instance = injector.instantiate(['a', 'b', Type]);
  expect(instance.result).toBe(3);
});
```

最后是推断式依赖注入：

test/injector_spec.js

```js
it('instantiates a non-annotated constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  function Type(a, b) {
    this.result = a + b;
  }
  var instance = injector.instantiate(Type);
  expect(instance.result).toBe(3);
});
```

我们先把 instantiate 接口暴露出来：

src/injector.js

```js
return {
  has: function(key) {
    return cache.hasOwnProperty(key);
  },
  get: function(key) {
    return cache[key];
  },
  annotate: annotate,
  invoke: invoke,
  instantiate: instantiate
};
```

我们先实现 instantiate 的简单版，我们先生成一个对象，然后使用这个对象作为上下文调用 invoke，最后返回这个对象：

src/injector.js

```js
function instantiate(Type) {
  var instance = {};
  invoke(Type, instance);
  return instance;
}
```

现在上面的测试用例都通过了，但我们漏掉了使用 new 实例化对象的一个特性——继承原型链，为了确保完整性，我们也需要实现这一特性：

```js
it('uses the prototype of the constructor when instantiating', function() {
  function BaseType() {}
  BaseType.prototype.getValue = _.constant(42);

  function Type() {
    this.v = this.getValue();
  }
  Type.prototype = BaseType.prototype;
  var module = window.angular.module('myModule', []);
  var injector = createInjector(['myModule']);
  var instance = injector.instantiate(Type);
  expect(instance.v).toBe(42);
});
```

为了设置原型链，我们要改用 ES5.1 的 Object.create 方法来创建对象。另外，在行内式注入时，我们也要把最后一个数组元素抽取出来，这个元素才是我们需要的注射器函数：

src/injector.js

```js
function instantiate(Type) {
  var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
  var instance = Object.create(UnwrappedType.prototype);
  invoke(Type, instance);
  return instance;
}
```

当然，因为我们实例化的过程中调用了 invoke 方法，我们也需要实现 invoke 附带本地变量的特性，我们可以把本地变量作为 instantiate 的第二个参数传入，并在 invoke 时使用：

test/injector_spec.js

```js
it('supports locals when instantiating', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  function Type(a, b) {
    this.result = a + b;
  }
  var instance = injector.instantiate(Type, {
    b: 3
  });
  expect(instance.result).toBe(4);
});
```

src/injector.js

```js
function instantiate(Type, locals) {
  var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
  var instance = Object.create(UnwrappedType.prototype);
  invoke(Type, instance, locals);
  return instance;
}
```
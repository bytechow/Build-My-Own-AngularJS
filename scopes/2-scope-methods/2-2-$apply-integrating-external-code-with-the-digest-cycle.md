### $apply——把外部代码集成到 digest 循环中
#### $apply - Integrating External Code With The Digest Cycle

作用域对象上最广为人知的函数应该是 `$apply`。它被看作是把外部代码库集成到 Angular 框架的标准范式，这是有原因的。

`$apply` 会接受一个函数作为参数，然后在内部使用 `$eval` 执行这个函数，最后会调用 `$digest` 来主动启动 digest 循环。下面是一个与此相关的单元测试：

_test/scope_spec.js_

```js
describe('$apply', function() {

  var scope;
  
  beforeEach(function() {
    scope = new Scope();
  });

  it('executes the given function and starts the digest', function() {
    scope.aValue = 'someValue';
    scope.counter = 0;

    scope.$watch(
      function(scope) {
        return scope.aValue;
      },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );
    
    scope.$digest();
    expect(scope.counter).toBe(1);
    
    scope.$apply(function(scope) {
      scope.aValue = 'someOtherValue';
    });
    expect(scope.counter).toBe(2);
  });

});
```

我们设置了一个 watcher 用于侦听 `scope.aValue`，然后在它发生变化时让计数器自增 1。我们要验证的是，在调用 `$apply` 后，watcher 是否会被马上执行：

下面是一个简化版的 `$apply` 方法，这样就能通过上面的测试了：

_src/scope.js_

```js
Scope.prototype.$apply = function(expr) {
  try {
    return this.$eval(expr);
  } finally {
    this.$digest();
  }
};
```

我们会在 `finally` 代码块中调用 `$digest`，这样才能保证即使执行函数时发生了异常，digest 循环依然会被启动。

`$apply` 的核心思想就是让我们可以执行一些 Angular 体系以外的代码。这些代码依然能够改变作用域上的数据，只要我们把这些代码包裹在 `$apply` 中，就能保证作用域上的 watcher 能捕捉到这些变更。当有人谈起使用 `$apply` 将代码集成到 Angular d的生命周期中时，这就是他们真正的意思，没有什么比这更重要的了。
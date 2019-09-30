### $apply——把外部代码和 digest 循环结合到一起
#### $apply - Integrating External Code With The Digest Cycle

作用域对象上最广为人知的函数应该是 `$apply`。它被看作是把外部代码库集成到 Angular 框架的标准方式，肯定是有原因的。

`$apply` 需要传入一个函数作为参数，而在内部使用 `$eval` 来执行这个函数，然后就使用 `$digest` 方法来触发一个 digest 周期。下面就是一个与此相关的单元测试：

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

我们设置了一个 watcher 用于侦听 `scope.aValue`，然后在它发生变化时让计数器增加。我们验证在调用 `$apply` 之后，watcher （的 listener 函数）是否会被执行：

下面是一个简单版的 `$apply`，这就能让我们上面的测试通过了：

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

我们会在 `finally` 代码块中调用 `$digest`，这样才能保证即使提供的函数会抛出异常，digest 也会被正常触发。

`$apply` 的核心概念就是让我们可以执行一些 Angular 体系以外的代码。这些代码依然可以改变作用域上的数据，而只要我们把这些代码包裹在 `$apply` 中，就能保证作用域上的 watcher 能捕捉到这些变更。当有人谈起使用 `$apply` 集成外部代码到 Angular 时，实际上他们想表达的就是这个意思，并没有什么其他的特殊含义。
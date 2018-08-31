### 构建一个可配置的provider：Digest TTL（Making a Configurable Provider: Digest TTL）

由于我们现在已经使用 provider 的方式对 $rootScope 进行封装，我们就可以对其进行一些配置。在本书第一章，我们引入了 digest TTL 的概念，digest TTL 指的是在我们在脏值检查中发现状态并不固定时再次检查的次数上限，如果超出该上限，我们就抛出异常。之前我们设置的 TTL 值是 10，但我们应该允许使用 provider 配置的方式对这个值进行更改。

实际上，AngularJS 应用开发者可以通过注入 $rootScopeProvider 的方式来对 TTL 进行配置。下面我们会在 scope_spec.js 中新增一个测试模块：

```js
describe('TTL configurability', function() {
  beforeEach(function() {
    publishExternalAPI();
  });
  it('allows configuring a shorter TTL', function() {
    var injector = createInjector(['ng', function($rootScopeProvider) {
      $rootScopeProvider.digestTtl(5);
    }]);
    var scope = injector.get('$rootScope');
    scope.counterA = 0;
    scope.counterB = 0;
    scope.$watch(
      function(scope) {
        return scope.counterA;
      },
      function(newValue, oldValue, scope) {
        if (scope.counterB < 5) {
          scope.counterB++;
        }
      }
    );
    scope.$watch(
      function(scope) {
        return scope.counterB;
      },
      function(newValue, oldValue, scope) {
        scope.counterA++;
      }
    );
    expect(function() {
      scope.$digest();
    }).toThrow();
  });
});
```

上面的测试用例，我们依然会有两个独立的 watcher，但是我们将 TTL 设置为 5，然后通过逻辑处理，控制当 TTL 为 10 时不会抛出错误。然而，当检查次数超过超出 5 次时会抛出错误。

那么，接下来，我们就需要在 $rootScopeProvider 中加入这个配置方法，注意，TTL 的默认值仍然是 10：

```js
function $RootScopeProvider() {
  var TTL = 10;
  this.digestTtl = function(value) {
    if (_.isNumber(value)) {
      TTL = value;
    }
    return TTL;
  };
  // ...
}
```

注意，digestTtl 方法也可以用于获取 TTL 的值。

在 $digest 中，我们需要换成使用常量 TTL 作为初始值，而不是 hard-coded 的 10：

```js
Scope.prototype.$digest = function() {
  var ttl = TTL;
  var dirty;
  this.$root.$$lastDirtyWatch = null;
  this.$beginPhase('$digest');
  if (this.$root.$$applyAsyncId) {
    clearTimeout(this.$$applyAsyncId);
    this.$$flushApplyAsync();
  }
  do {
    while (this.$$asyncQueue.length) {
      try {
        var asyncTask = this.$$asyncQueue.shift();
        asyncTask.scope.$eval(asyncTask.expression);
      } catch (e) {
        console.error(e);
      }
    }
    dirty = this.$$digestOnce();
    if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
      throw TTL + ' digest iterations reached';
    }
  } while (dirty || this.$$asyncQueue.length);
  this.$clearPhase();
  while (this.$$postDigestQueue.length) {
    try {
      this.$$postDigestQueue.shift()();
    } catch (e) {
      console.error(e);
    }
  }
};
```

从上面这个例子，我们也可以看到抽象出 provider 的作用。作为应用开发者，我们可以配置 $rootScope，而无须修改它的源码，也不需要在装饰器中重写某些东西。$rootScopeProvider 提供了一个 API 来配置 $rootScope TTL。应用开发者如果在开发过程中发现确实需要超过 10 个以上的 watch 依赖，可以设置一个大于 10 的值。当然，也允许应用开发者设置一个小于 10 的值来对性能进行优化。
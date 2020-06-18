### 发现值始终无法稳定下来时结束 digest（Giving Up On An Unstable Digest）

目前我们的代码实现中还遗漏了一种情况没有处理：如果两个 watcher 都在监听着会被对方更改的值会发生什么？换言之，如果属性状态永远无法稳定下来我们该怎么办？下面这个单元测试就展示了这种情况：

_test/scope_spec.js_

```js
it('gives up on the watches after 10 iterations', function() {
  scope.counterA = 0;
  scope.counterB = 0;
  
  scope.$watch(
    function(scope) { return scope.counterA; },
    function(newValue, oldValue, scope) {
      scope.counterB++;
    }
  );
  
  scope.$watch(
    function(scope) { return scope.counterB; },
    function(newValue, oldValue, scope) {
      scope.counterA++;
    }
  );

  expect((function() { scope.$digest(); })).toThrow();
});
```

对于这种情况，我们希望在调用 `scope.$digest` 之后会抛出一个异常，但事实并没有如我们所愿。事实上，这个测试用例将会一直执行下去无法结束。这是因为这里的两个计数器变量都相互依赖又相互影响，一轮 `$$digestOnce` 中这两个变量总有一个会变 “脏”。

> 注意，我们在调用 Jasmine `expect` 函数时没有直接调用 `scope.$digest`，而是传递了一个函数，然后这个函数再调用 `scope.$digest`。这样 `expect` 才能检查调用 `scope.$digest` 时是否会抛出异常。

> 由于这个测试不会自动结束，你需要先杀掉 karma 进程，在解决这个问题之后再重启 Karma 就可以了。

我们需要把 digest 的运行轮次控制在一个合理的数值以内。如果 digest 运行次数已经超过了这个上限，但 scope 仍有变化发生，我们就会放弃继续进行 digest 并宣告它可能永远没法进入稳定状态。此时，我们不妨抛出一个异常，因为无论 scope 当前的状态数据是什么，它都不太可能是用户想要的。

这个最大迭代次数被称为 TTL（“Time To Live”的简写）。它的默认值是 10。这个数字看上去很小，但请记住，这里对性能十分敏感，它会被频繁运行，且每次运行都会把所有 watch 函数执行一遍。用户也不大可能搞出 10 个连在一起的 watcher。

> 实际上，Angular 是允许调整 TTL 数值的。后面在讲到 provider 和依赖注入时，我们会具体介绍如何进行调整。

下面，我们在外层的 digest 循环中加入一个循环计数器，一旦循环次数达到 TTL 上限就抛出异常：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  do {
    dirty = this.$$digestOnce();
    if (dirty && !(ttl--)) {
      throw '10 digest iterations reached';
    }
  } while (dirty);
};
```

这个升级版的 `$digest` 会在发生循环依赖时抛出异常，这跟我们的单元测试预期的一样。这能避免 digest 进行死循环。 
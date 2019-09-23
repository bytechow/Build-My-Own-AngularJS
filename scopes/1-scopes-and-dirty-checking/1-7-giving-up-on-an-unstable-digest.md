### 发现值始终无法稳定下来时结束 digest（Giving Up On An Unstable Digest）

在我们目前的代码实现中还有着一个明显的遗漏情况：如果两个 watcher 都在监听着由对方造成变化的属性会发生什么？也就是说，如果属性状态永远无法稳定下来该怎么办？我们通过以下单元测试来展示这种情况：

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

我们希望调用 `scope.$digest` 时会抛出一个异常，但事实上并没有如我们所愿。事实上，这个测试将一直无法结束。这是因为里面的两个计数器变量相互依赖、影响，所以在每次执行 `$$digestOnce` 的时候它们其中之一就会变 “脏”。

> 注意我们没有直接调用 `scope.$digest`，而是在调用 Jasmine `expect` 函数时传递了一个函数（在函数中调用 `scope.$digest`）。`expect` 函数会替我们调用这个函数，这样才能检查调用 `scope.$digest` 时是否会抛出异常。

> 由于这个测试不能自动结束，你需要杀掉 karma 进程，并在我们解决这个问题后再重启。

我们需要做的事是让 digest 的运行次数限制在合理数量以内。如果 digest 运行了约定的最大迭代次数以后，scope 仍有变化发生，我们就要放弃继续运行 digest 并说明此次 digest 可能永远没法进入稳定状态。此时，我们最好抛出异常，因为无论此时作用域处于何种状态，都不太可能是用户想要的。

最大迭代次数被称为 TTL（“Time To Live”的简写）。它的默认值是 10。这个数字看上去很小，但要知道 digest 会经常运行，且每次运行都会把所有 watch 函数执行一遍，因此这是一个性能敏感区域。而且也不大可能出现 10 个以上、构成循环依赖的 watcher。

> Angular 中实际上是允许调整 TTL 数值的。后面我们讲到 provider 和依赖注入时会介绍如何进行调整。

下面我们再外层的 digest 循环中加入一个循环计数器。如果循环次数达到 TTL 上限，则抛出异常：

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

这个升级版的 `$digest` 能在发生循环依赖时抛出异常，就像我们的单元测试所希望的那样。这能避免 digest 运行陷入无限循环。 
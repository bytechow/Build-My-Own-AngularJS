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
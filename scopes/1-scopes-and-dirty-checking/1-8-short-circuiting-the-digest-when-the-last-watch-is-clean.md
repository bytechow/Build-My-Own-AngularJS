### 如果上轮最后一个变脏的 watcher 未发生变化则绕过本轮 digest（Short-Circuiting The Digest When The Last Watch Is Clean）

在目前的代码中，我们会对作用域中 watcher 集合进行持续遍历，直到在某轮遍历中所有 watcher 都未发生变化（或者达到了 TTL 上限）。

鉴于一个 digest 循环可能包含大量的 watcher，尽可能地降低执行 watcher 的次数是非常重要的。这也是为什么我们要对 digest 循环使用一种特有的优化手段。

加入一个作用域上有 100 个 watcher。当我们在作用域上启动 digest 后发现只有第一个 watcher 是“脏”的。也就是说这一个 watch 把这一轮 digest 弄“脏”了，我们不得不再执行一轮 digest。在第二轮中，我们发现没有 watcher 变“脏”后才结束 digest。但这意味着在结束之前，我们需要执行 200 个 watcher！

有一种方法可以让执行量减半，就是对上一个发生变化的 watcher 进行记录。然后，每当我们遇到一个没有发生变化 watcher，我们就看它跟我们记录的 watcher 是否是同一个。如果发现是同一个，说明这一轮已经没有 watcher 变脏了，证明我们没有必要继续执行本轮剩余的 watcher 了，可以马上退出本轮 digest 了。下面的单元测试就是针对这种情况的：

_test/scope_spec.js_

```js
it('ends the digest when the last watch is clean', function() {
  scope.array = _.range(100);
  var watchExecutions = 0;

  _.times(100, function(i) {
    scope.$watch(
      function(scope) {
        watchExecutions++;
        return scope.array[i];
      },
      function(newValue, oldValue, scope) {}
    );
  });
  
  scope.$digest();
  expect(watchExecutions).toBe(200);
  
  scope.array[0] = 420;
  scope.$digest();
  expect(watchExecutions).toBe(301);
});
```
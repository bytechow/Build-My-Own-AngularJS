### 在 watch 函数中设置 $evalAsync 定时任务
#### Scheduling $evalAsync from Watch Functions

上一节我们说可以在 listener 函数可以用 `$evalAsync` 来设定一个延时任务，这个任务依然会在同一个 digest 周期中被执行。但如果我们是在 watch 函数内用 `$evalAsync` 设定一个延时任务会发生什么呢？当然，我们不推荐这样做，因为我们认为 watch 函数不应该产生任何副作用。但在 Angular 这种做法也是被允许的，所以我们要保证在这种情况下不会对 digset 产生不良的影响。

如果我们在 watch 函数调用一次 `$evalAsync`，看上去是运行正常的。像下面这个单元测试，我们不需要修改代码就能让它通过了：

_test/scope\_spec.js_

```js
it('executes $evalAsynced functions added by watch functions', function() {
  scope.aValue = [1, 2, 3];
  scope.asyncEvaluated = false;

  scope.$watch(
    function(scope) {
      if (!scope.asyncEvaluated) {
        scope.$evalAsync(function(scope) {
          scope.asyncEvaluated = true;
        });
      }
      return scope.aValue;
    },
    function(newValue, oldValue, scope) { }
  );

  scope.$digest();

  expect(scope.asyncEvaluated).toBe(true);
});
```

那究竟问题出在哪里呢？正如我们所见，只要还有一个 watch 是“脏”的，我们就会继续进行 digest 循环。在上面的测试用例中，在第一轮 digest 遍历时，我们首次从 watch 函数中返回 `scope.aValue`，这会触发第二轮 digest 循环。这第二轮 digest 开始时，会先执行在 watch 中设定的 `$evalAsync` 延迟任务。但如果在没有 watcher 变“脏”的情况下调用 `$evalAsync` 又该怎么处理呢？

_test/scope\_spec.js_

```js
it('executes $evalAsynced functions even when not dirty', function() {
  scope.aValue = [1, 2, 3];
  scope.asyncEvaluatedTimes = 0;

  scope.$watch(
    function(scope) {
      if (scope.asyncEvaluatedTimes < 2) {
        scope.$evalAsync(function(scope) {
          scope.asyncEvaluatedTimes++;
        });
      }
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {}
  );

  scope.$digest();

  expect(scope.asyncEvaluatedTimes).toBe(2);
});
```

这个版本的测试用例会调用两次 `$evalAsync`。在第二次调用时，由于 `scope.aValue` 已经不再变化，watcher 也就不会变脏了。这意味着 `$digest` 结束了，而（在第二轮 digest 循环中）`$evalAsync` 设定的延时不会被调用了。虽然在它能在下一次 digest 时被执行，但我们希望它在当前 digest 中就能执行。这就意味着我们需要改变 `$digest` 循环的终止条件，如果在异步任务队列中还存在任务，就继续循环：

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  // do {
  //   while (this.$$asyncQueue.length) {
  //     var asyncTask = this.$$asyncQueue.shift();
  //     asyncTask.scope.$eval(asyncTask.expression);
  //   }
  //   dirty = this.$$digestOnce();
  //   if (dirty && !(ttl--)) {
  //     throw '10 digest iterations reached';
  //   }
  } while (dirty || this.$$asyncQueue.length);
};
```

这样这个单元测试就通过了，但现在我们介绍另一个存在的问题。如果一个 watch 函数一直使用 `$evalAsync` 来设置延迟任务又该怎么办？我们希望在这种情况下迭代会达到上限，但事实并不是这样的：

_test/scope\_spec.js_

```js
it('eventually halts $evalAsyncs added by watches', function() {
  scope.aValue = [1, 2, 3];

  scope.$watch(
    function(scope) {
      scope.$evalAsync(function(scope) { });
      return scope.aValue;
    },
     function(newValue, oldValue, scope) { }
  );

  expect(function() { scope.$digest(); }).toThrow();
});
```

这个单元测试会一直运行着，因为 `$digest` 里的 `while` 循环一直都不能结束。我们需要做的就是在对 TTL 的检查中，加入对异步任务队列的验证：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  // do {
  //   while (this.$$asyncQueue.length) {
  //     var asyncTask = this.$$asyncQueue.shift();
  //     asyncTask.scope.$eval(asyncTask.expression);
  //   }
  //   dirty = this.$$digestOnce();
    if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty || this.$$asyncQueue.length);
};
```

这样我们就能保证单元测试的 digest 会结束了，无论是因为还有 watcher 变“脏”还是因为异步任务队列还有任务存在。


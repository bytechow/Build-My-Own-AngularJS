### 在 watch 函数中设置 $evalAsync 定时任务
#### Scheduling $evalAsync from Watch Functions

上一节我们说可以在 listener 函数可以用 `$evalAsync` 来设定一个延时任务，这个任务依然会在同一个 digest 周期中被执行。但如果我们是在 watch 函数设定 `$evalAsync` 延时任务会发生什么呢？当然，我们不推荐这样做，因为我们认为 watch 函数不应该产生任何的副作用。但在 Angular 这种做法也是被允许的，所以我们要保证这种操作下不会对 digset 产生不良的影响。

如果我们在 watch 函数调用一次 `$evalAsync`，结果看上去是满足我们需求的。像下面这个单元测试，我们不需要修改代码就能让它通过了：

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

那究竟问题出在哪里呢？正如我们所见，只要还有一个 watch 函数变“脏”的，我们就会继续进行 digest 循环。在上面的测试用例中，在第一轮 digest 遍历时，我们首次从 watch 函数中返回 `scope.aValue`，这会触发第二轮 digest 循环。这第二轮 digest 自然就会顺带着执行在 watch 中设定的 `$evalAsync` 延迟任务。但如果没有 watcher 变“脏”的情况下设定 `$evalAsync` 延时任务又该怎么处理呢？

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

这次我们会调用两次 `$evalAsync`。在第二次调用时，由于 `scope.aValue` 不再发生变化，watch 函数是“干净”的。这也意味着 `$digest` 会被终止，这个 `$evalAsync` 设定的延时任务也就不会被执行了。虽然它可以在下一次 digest 周期被执行，但我们希望它能够在当前 digest 周期执行。这就意味着我们需要改变 `$digest` 循环的终止条件，只要异步任务队列中还存在任务，我们就继续循环：

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

这样虽然能通过上面的单元测试，但也会引发一个新的问题。如果一个 watch 函数一直使用 `$evalAsync` 来设置延迟任务该怎么办？我们希望在这种情况下迭代也会有一个上限值，但事实上并没有：

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

这个单元测试会一直运行，因为 `$digest` 里的 `while` 循环不会被结束。我们需要在 TTL 检查中加入对异步任务队列的检查：

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

这样我们就能保证无论是因为 watch 函数 变“脏”，还是因为异步任务队列还有任务存在，我们都能确保 digest 周期会在达到迭代上限时被终止。


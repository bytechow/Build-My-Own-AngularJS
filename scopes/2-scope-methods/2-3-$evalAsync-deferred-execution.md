### $evalAsync——延迟执行
#### $evalAsync - Deferred Execution

在 JavaScript 中，要延迟执行一些代码的情况是很常见的——让这段代码延迟到当前执行上下文结束时再执行。一般的做法是使用 [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout) 方法，传入的 delay 参数为 0。

我们也会在 Angular 应用这种模式。虽然除开其他因素，我们更推荐的方式是使用 `$timeout` 服务进行定时，然后利用 `$apply` 把要延迟执行的函数融入到 digest 周期中。

但在 Angular 中还有一种方式可以支持延迟执行代码，也就是作用域上的 `$evalAsync` 函数。`$evalAsync` 会接收一个函数，然后让这个函数延迟执行，但函数的实际执行时间仍然会在当前正在进行的 digest 周期中。举个例子，你可以在一个 watcher 的 listener 函数中设定延迟执行一段代码，而且你可以确认虽然这段代码被延迟执行了，但还是会在当前的 digest 周期中被调用。

与传入的 delay 参数为 0 的 `$timeout` 服务相比，`$evalAsync` 更有优势的原因与浏览器的事件循环机制（event loop）有关。当你使用 `$timeout` 设定定时任务时，你就会需要把控制器归还给浏览器，由浏览器来决定这个定时任务实际的执行时间。浏览器在执行这个定时任务之前，可能会选择去执行其他任务。这些任务可能是渲染 UI，也可能是运行点击事件处理函数，或者是处理 Ajax 响应。另一方面，`$evalAsync`队定时任务的执行时间要严格得多。由于 `$evalAsync` 设定的定时任务会在当前运行的 digest 周期中执行，也就相当于保证它会在浏览器执行其他任务之前执行。`$timeout` 和 `$evalAsync` 的这个不同之处，在我们想要避免不必要的渲染的情况下尤为关键：为什么要让浏览器对之后马上就会发生变化的 DOM 进行渲染呢？

我们用一个单元测试来说明 `$evalAsync` 函数签名：

_test/scope_spec.js_

```js
describe('$evalAsync', function() {
  
  var scope;
  
  beforeEach(function() {
    scope = new Scope();
  });
  
  it('executes given function later in the same cycle', function() {
    scope.aValue = [1, 2, 3];
    scope.asyncEvaluated = false;
    scope.asyncEvaluatedImmediately = false;
    
    scope.$watch(
      function(scope) { return scope.aValue; },
      function(newValue, oldValue, scope) {
        scope.$evalAsync(function(scope) {
          scope.asyncEvaluated = true;
        });
        scope.asyncEvaluatedImmediately = scope.asyncEvaluated;
      }
    );

    scope.$digest();
    expect(scope.asyncEvaluated).toBe(true);
    expect(scope.asyncEvaluatedImmediately).toBe(false);
  });
  
});
```

我们在 watcher 的 listener 函数中调用了 `$evalAsync` 函数，然后验证这个函数是否在同一个 digest 周期中被执行，且执行时间是在 listener 函数结束之后。

我们需要做的第一件事是新建一个空间来存储 `$evalAsync` 要执行的定时任务。我们可以在 Scope 构造函数中初始化一个数组作为定时任务队列的存储空间：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
  this.$$lastDirtyWatch = null;
  this.$$asyncQueue = [];
}
```

然后我们定义 `$evalAsync` 方法，它会把要定时执行的函数存放到队列中去：

_src/scope.js_

```js
Scope.prototype.$evalAsync = function(expr) {
  this.$$asyncQueue.push({scope: this, expression: expr});
};
```

> 我们把作用域本身也保存到队列中是有原因的，下一章会对此进行介绍。

我们已经对定时任务进行了保存，但还没有实际执行它们。定时任务会在 `$digest` 方法中执行：`$digest` 方法最先做的一件事就是对定时任务队列进行遍历，依次使用对应作用域上的 `$eval` 方法执行定时任务：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  // do {
    while (this.$$asyncQueue.length) {
      var asyncTask = this.$$asyncQueue.shift();
      asyncTask.scope.$eval(asyncTask.expression);
    }
  //   dirty = this.$$digestOnce();
  //   if (dirty && !(ttl--)) {
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty);
};
```

这样我们就能保证定时任务不会被马上执行，而且当作用域还是“脏”的，这个定时任务就会在当前 digest 周期中执行。这样就满足了我们的需求了。
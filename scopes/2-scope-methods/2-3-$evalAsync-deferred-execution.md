### $evalAsync——延迟执行

#### $evalAsync - Deferred Execution

在 JavaScript 中，延迟执行一段代码是很常见的——让这段代码延迟到当前执行上下文完成后的某个时间点再执行。常用的方法是调用 [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout) API，并传入 0 \(或非常小的数值）作为参数。

我们也会在 Angular 应用程序用到这种模式，首选的方式是使用 `$timeout` 服务，它会使用 `$apply` 把延迟执行的函数集成到 digest 循环中。

但在 Angular 中还有一种方式可以支持延迟执行代码，那就是 scope 对象上的 `$evalAsync` 方法。`$evalAsync` 会接收一个函数，然后让这个函数延期执行，但只会延期到当前 digest 周期中某个时刻，不会更晚了。比如，我们就可以在一个 watcher 的 listener 函数中设定延迟执行某段代码，因为我们知道虽然这段代码会延期执行了，但执行时间依然会在当前的 digest 周期中，所以不会产生问题。

鉴于浏览器存在的事件循环机制（event loop），`$evalAsync` 比延迟参数为 0 的 `$timeout` 服务更有优势。当我们使用 `$timeout` 设定定时任务时，会把控制权交还给浏览器，然后就由浏览器来决定何时执行这些延期任务。浏览器在执行这些延期任务之前，可能会选择先去执行其他任务，可能是渲染 UI，也可能是运行点击事件的处理函数，又或者是处理 Ajax 响应。另一方面，`$evalAsync` 对定时任务的执行时间要严格得多。`$evalAsync` 设定的定时任务必须在当前运行的 digest 周期中执行，也就能保证这个任务会在浏览器执行其他任务之前被执行。在我们想要避免浏览器不必要的渲染时，`$timeout` 和 `$evalAsync` 之间的差异就更明显了：为什么要让浏览器渲染那些马上会被覆盖的 DOM 更改呢？

我们用一个单元测试来定义 `$evalAsync` 函数：

_test/scope\_spec.js_

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

我们在 watcher 的 listener 函数中调用了 `$evalAsync` 函数，然后验证这个函数是否在同一个 digest 周期中被执行，且执行时间是在 listener 函数调用结束之后。

我们要做的第一件事是提供一个空间来存储 `$evalAsync` 要执行的定时任务。我们可以在 Scope 构造函数中初始化一个数组属性成员作为定时任务的存储空间：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
  this.$$lastDirtyWatch = null;
  this.$$asyncQueue = []; // +
}
```

下一步，我们要定义 `$evalAsync` 方法，它会把要定时执行的函数存放到队列中去：

_src/scope.js_

```js
Scope.prototype.$evalAsync = function(expr) {
  this.$$asyncQueue.push({scope: this, expression: expr});
};
```

> 我们把当前作用域也保存到任务队列是有原因的，下一章会说到。

虽然我们保存了要执行的定时任务，但还没有去执行它们。定时任务会在 `$digest` 周期中被执行：`$digest` 方法最先要做的一件事就是对定时任务队列进行遍历，然后依次使用对应作用域的 `$eval` 方法执行各个定时任务：

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

这样我们就能保证定时任务不会被马上执行，而且当作用域还是“脏”的，这个定时任务就会在当前 digest 周期内被执行。这就满足我们的需求了。


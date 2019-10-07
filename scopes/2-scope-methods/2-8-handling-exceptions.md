### 异常处理
#### Handling Exceptions

我们之前开发的 `$evalAsync`、`$applyAsync` 和 `$$postDigest` 都存在一个问题，就是当异常发生时，函数的执行进程会被终止，而且 digest 循环也会被迫过早结束。然而，真正的 Angular 的实现方式比现在的健壮得多。无论是在 digest 之前、之中或之后抛出的异常都能被捕捉到，并记入日志。

对于 `$evalAsync`，我们可以定义一个测试用例（在 `describe('$evalAsync')` 的测试板块中），在这个测试中，当其中一个由 `$evalAsync` 设定为延时执行的函数抛出异常时，我们验证 watcher 是否依然会被执行。

_test/scope_spec.js_

```js
it('catches exceptions in $evalAsync', function(done) {
  scope.aValue = 'abc';
  scope.counter = 0;

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$evalAsync(function(scope) {
    throw 'Error';
  });
  
  setTimeout(function() {
    expect(scope.counter).toBe(1);
    done();
  }, 50);
});
```

而对于 `$applyAsync` 的单元测试，我们会使用 `$applyAsync` 设定几个延时执行的函数，当前面的函数抛出异常，我们验证最后的那个函数是否能被正常调用。这会放到 `describe('$applyAsync')` 的测试板块中。

_test/scope_spec.js_

```js
it('catches exceptions in $applyAsync', function(done) {
  scope.$applyAsync(function(scope) {
    throw 'Error';
  });
  scope.$applyAsync(function(scope) {
    throw 'Error';
  });
  scope.$applyAsync(function(scope) {
    scope.applied = true;
  });
  
  setTimeout(function() {
    expect(scope.applied).toBe(true);
    done();
  }, 50);
});
```

> 注意，这里我们连续用了两个会抛出异常的函数，因为如果我们只用一个的话，第二个函数本来就会被运行了。这是因为 `$apply` 函数中的 `finally` 代码块中会触发 `$digest`，在这个 `$digest` 中，`$applyAsync` 创建的异步任务队列就会被执行完毕。
>
> （译者注：上面的解释比较简略，这里详细说下流程。如果我们只用一个会抛出异常的函数，那就是 `$applyAsync` 一共会设定两个函数，一个会抛异常，另一个不会。第一个 `$applyAsync` 函数会设定一个定时器，定时器到时执行 `$apply`，`$apply` 会利用 `$eval` 调用 `$$flushApplyAsync`，`$$flushApplyAsync` shift 出第一个异步任务并调用，会抛出异常，导致 `$$flushApplyAsync` 调用过程发生中断，返回到上层也就是 `$apply`，由于 `$apply` 使用了 `try...finally` 代码块对 `$eval` 进行包裹，因此异常会被忽略，并进入到 `finally` 代码块，`finally` 代码块中调用了 `$digest`，这样剩余的一个异步任务就会在 `$digest` 函数处理 `$applyAsync` 设定的异步任务时被调用了）

由于 `$$postDigest` 发生在 digest 结束之后，就不能再使用 watcher 来测试了。但我们可以使用第二个 `$$postDigest` 函数来测试，确保这个函数会被执行。我们把这个测试放到 `describe('$$postDigest')` 测试板块中。

_test/scope_spec.js_

```js
it('catches exceptions in $$postDigest', function() {
  var didRun = false;
  
  scope.$$postDigest(function() {
    throw 'Error';
  });
  scope.$$postDigest(function() {
    didRun = true;
  });

  scope.$digest();
  expect(didRun).toBe(true);
});
```

解决 `$evalAsync` 和 `$$postDigest` 存在的异常处理问题与 `$digest` 函数有关。我们只需要在 `$digest` 对应的位置包裹上 `try...catch` 代码块即可：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  // this.$beginPhase('$digest');
  
  // if (this.$$applyAsyncId) {
  //   clearTimeout(this.$$applyAsyncId);
  //   this.$$flushApplyAsync();
  // }

  do {
    while (this.$$asyncQueue.length) {
      try {
        // var asyncTask = this.$$asyncQueue.shift();
        // asyncTask.scope.$eval(asyncTask.expression);
      } catch (e) {
        console.error(e);
      }
    }
    // dirty = this.$$digestOnce();
    // if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
    //   throw '10 digest iterations reached';
    // }
  } while (dirty || this.$$asyncQueue.length);
  // this.$clearPhase();

  while (this.$$postDigestQueue.length) {
    try {
      // this.$$postDigestQueue.shift()();
    } catch (e) {
      console.error(e);
    }
  }
};
```

而对于 `$applyAsync`，我们就需要在 `$$flushApplyAsync` 对异步任务进行遍历时进行处理：

_src/scope.js_

```js
Scope.prototype.$$flushApplyAsync = function() {
  while (this.$$applyAsyncQueue.length) {
    try {
      // this.$$applyAsyncQueue.shift()();
    } catch (e) {
      console.error(e);
    }
  }
  // this.$$applyAsyncId = null;
};
```

现在我们的 digest 循环在异常处理方面比之前健壮多了。
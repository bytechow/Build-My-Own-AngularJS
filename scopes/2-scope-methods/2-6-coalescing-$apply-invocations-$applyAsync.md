### 合并 $apply 调用—— $applyAsync

#### Coalescing $apply Invocations - $applyAsync

虽然 `$evalAsync` 既可以在 digest 内设置延时任务，也可以在 digest 外设置延时任务，但实际上它就是为前者设计的。如果有人在 digest 外调用了 `$evalAsync`，那 `$evalAsync` 内部调用 `setTimeout` 主要是为了防止混淆。

而对于要在 digest 循环外异步调用（apply）一个函数的情况，Angular 提供了另一个专用函数 `$applyAsync`。它的作用与 `$apply` 差不多——都是用于将无法感知到 Angular digest 周期的外部代码进行集成。但跟 `$apply` 不同的是，`$applyAsync` 不会马上调用传入的函数，而且也不会立即触发一个 digest。相反，`$applyAsync` 会推迟一段比较短的时间之后再执行这两个操作。

`$applyAsync` 最初的设计动机是用于处理 HTTP 响应：当 `$http` 服务收到一个响应之后，就会调用对应的处理函数并且启动一个 digest。这意味着每次 HTTP 响应时都要重新运行一个 digest。这样的话，如果应用的 HTTP 通信比较频繁或者它的 digest 循环需要大量计算时（很多应用启动时都可能会遇到这种情况），就会产生性能问题。目前，`$http` 可以配置为使用 `$applyAsync` 模式，它会将返回时间非常接近的 HTTP 响应都合并到一个 digest 内进行处理。但是，`$applyAsync` 不仅仅只能用于 `$http` 服务进行请求，你还可以把它应用到任何可以从“合并到一个 digest 进行处理“的模式获益的情况中去。

在 `$applyAsync` 的第一个测试用例中，我们希望调用函数后不会立马发生改变，而会在 50 毫秒以后才会产生变化：

_test/scope\_spec.js_

```js
describe('$applyAsync', function() {

  var scope;

  beforeEach(function() {
    scope = new Scope();
  });

  it('allows async $apply with $applyAsync', function(done) {
    scope.counter = 0;

    scope.$watch(
      function(scope) { return scope.aValue; },
      function(newValue, oldValue, scope) {
        scope.counter++;
      }
    );

    scope.$digest();
    expect(scope.counter).toBe(1);

    scope.$applyAsync(function(scope) {
      scope.aValue = 'abc';
    });
    expect(scope.counter).toBe(1);

    setTimeout(function() {
      expect(scope.counter).toBe(2);
      done();
    }, 50);
  });

});
```

目前我们还没有发现 `$applyAsync` 与 `$evalAsync` 有什么区别，但只要我们尝试在 listner 函数中调用 `$applyAsync` 就能发现端倪。如果我们在这里调用 `$evalAsync`，那么这个函数仍然会在同一个 digest 中被调用。但 `$applyAsync` 只会在 digest 结束后才进行调用：

_test/scope\_spec.js_

```js
it('never executes $applyAsync function in the same cycle', function(done) {
  scope.aValue = [1, 2, 3];
  scope.asyncApplied = false;

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.$applyAsync(function(scope) {
        scope.asyncApplied = true;
      });
    }
  );

  scope.$digest();
  expect(scope.asyncApplied).toBe(false);
  setTimeout(function() {
    expect(scope.asyncApplied).toBe(true);
    done();
  }, 50);
});
```

开发 `$applyAsync` 的第一步是在 Scope 构造函数中添加另一个数组成员。这个数组就用于存放 `$applyAsync` 设定的异步任务：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  this.$$applyAsyncQueue = [];
  // this.$$phase = null;
}
```

每次调用 `$applyAsync`，我们都会把一个函数添加到这个队列中来。之后，这个函数就会在当前作用域的上下文内对传入的表达式进行运算，这跟 `$apply` 是一样的：

_src/scope.js_

```js
Scope.prototype.$applyAsync = function(expr) {
  var self = this;
  self.$$applyAsyncQueue.push(function() {
    self.$eval(expr);
  });
};
```

这里我们还需要设定用于调用这些函数的延时任务。我们可以直接用参数为零的 `setTimeout` 实现延时。在这个定时器函数中，我们会用 `$apply` 来触发一个函数，这个函数将会对 `$applyAsync` 对应的异步任务队列进行便利，并调用队列中的每一个函数：

_src/scope.js_

```js
Scope.prototype.$applyAsync = function(expr) {
  // var self = this;
  // self.$$applyAsyncQueue.push(function() {
  //   self.$eval(expr);
  // });
  setTimeout(function() {
    self.$apply(function() {
      while (self.$$applyAsyncQueue.length) {
        self.$$applyAsyncQueue.shift()();
      }
    });
  }, 0);
};
```

> 注意，我们不需要分别对队列中的每一个元素调用一次 `$apply`。我们只需要在循环以外调用一次 `$apply` 就可以了，我们只希望程序启动一次 digest。

如上所述，`$applyAsync` 的核心要点是对一些端时间内多次进行的操作进行优化，以便运行一次 digest 就能对这些操作带来的变化进行统一处理。我们现在还没有真正地实现这个目标。现在每次调用 `$applyAsync` 时都会设定一个启动 digest 的延时任务，我们只要在 watch 函数中对一个计数器进行自增就能明显地看出这个问题了：

_test/scope\_spec.js_

```js
it('coalesces many calls to $applyAsync', function(done) {
  scope.counter = 0;

  scope.$watch(
    function(scope) {
      scope.counter++;
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {}
  );

  scope.$applyAsync(function(scope) {
    scope.aValue = 'abc';
  });
  scope.$applyAsync(function(scope) {
    scope.aValue = 'def';
  });

  setTimeout(function() {
    expect(scope.counter).toBe(2);
    done();
  }, 50);
});
```

我们希望计数器的数值是 2，但不能大于这个数字。

> 译者注：因为在第一次 digest 中，因为第一轮循环肯定变“脏”，watch 会被执行两次。

我们需要使用一个私有的 scope 属性 `$$applyAsyncId` 来对执行异步任务队列的 `timeout` 定时器进行跟踪，看是不是它已经被设定好了：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  // this.$$applyAsyncQueue = [];
  this.$$applyAsyncId = null;
  // this.$$phase = null;
}
```

在设置延时任务之前，我们需要先检查一下这个属性，并在设置好定时器和延时任务完成之后对这个属性进行更新：

_src/scope.js_

```js
Scope.prototype.$applyAsync = function(expr) {
  // var self = this;
  // self.$$applyAsyncQueue.push(function() {
  //   self.$eval(expr);
  // });
  if (self.$$applyAsyncId === null) {
    self.$$applyAsyncId = setTimeout(function() {
      // self.$apply(function() {
      //   while (self.$$applyAsyncQueue.length) {
      //     self.$$applyAsyncQueue.shift()();
      //   }
        self.$$applyAsyncId = null;
    //   });
    // }, 0);
  }
};
```

`$applyAsync` 的另一个特性是，如果在它设定的 timeout 定时器触发之前由于其他某些原因已经启动了一个 digest，那定时器中的 digest 就无需启动了。这时，当前正在运行的 digest 就能将异步任务执行完，`$applyAsync` 的定时器就应该被销毁了：

_test/scope\_spec.js_

```js
it('cancels and flushes $applyAsync if digested first', function(done) {
  scope.counter = 0;

  scope.$watch(
    function(scope) {
      scope.counter++;
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {}
  );

  scope.$applyAsync(function(scope) {
    scope.aValue = 'abc';
  });
  scope.$applyAsync(function(scope) {
    scope.aValue = 'def';
  });

  scope.$digest();
  expect(scope.counter).toBe(2);
  expect(scope.aValue).toEqual('def');

  setTimeout(function() {
    expect(scope.counter).toBe(2);
    done();
  }, 50);
});
```

这里我们验证调用 `$digest` 后会不会把 `$applyAsync` 设定的任务都执行掉，之后就没有需要执行的任务了。

下面，我们会把 `$applyAsync` 中用于遍历执行异步任务队列的代码抽取成一个内部函数，这样我们就可以在多个地方重用它了：

_src/scope.js_

```js
Scope.prototype.$$flushApplyAsync = function() {
  while (this.$$applyAsyncQueue.length) {
    this.$$applyAsyncQueue.shift()();
  }
  this.$$applyAsyncId = null;
};

Scope.prototype.$applyAsync = function(expr) {
  // var self = this;
  // self.$$applyAsyncQueue.push(function() {
  //   self.$eval(expr);
  // });
  // if (self.$$applyAsyncId === null) {
  //   self.$$applyAsyncId = setTimeout(function() {
      self.$apply(_.bind(self.$$flushApplyAsync, self));
  //   }, 0);
  // }
};
```

> LoDash 的 `_.bind` 函数就相当于 ECMAScript 5 的 `Function.prototype.bind`，是用于指定返回函数的 this 值的。

现在，我们就可以在 `$digest` 函数中使用这个函数了，并判断如果当前已经有定时器处于待触发状态，我们就取消这个定时器并立即开始遍历异步任务队列：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  // this.$beginPhase('$digest');

  if (this.$$applyAsyncId) {
    clearTimeout(this.$$applyAsyncId);
    this.$$flushApplyAsync();
  }

  // do {
  //   while (this.$$asyncQueue.length) {
  //     var asyncTask = this.$$asyncQueue.shift();
  //     asyncTask.scope.$eval(asyncTask.expression);
  //   }
  //   dirty = this.$$digestOnce();
  //   if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty || this.$$asyncQueue.length);
  // this.$clearPhase();
};
```

这就是 `$applyAsync` 全部的内容了，它实际上就是 `$apply` 的“优化版”，在短时间内多次调用 `$apply` 时可以派上用场。


### 合并 $apply 调用—— $applyAsync
#### Coalescing $apply Invocations - $applyAsync

虽然 `$evalAsync` 既可以用于在一个 digest 内设置延时任务，也可以在 digest 外设置延时任务，但实际上它被设计出来主要是用于应付前一种情况的。digest 中调用的 `setTimeout` 主要只是为了防止有人在 digest 外调用 `$evalAsync` 而引起混乱。

而对于要在 digest 循环外异步地 apply 一个函数的情况，Angular 也有一个定制的函数 `$applyAsync`。它的作用与 `$apply` 差不多——都是用于将无法感知到 Angular digest 循环的外部代码整合进去。但跟 `$apply` 不同的是，`$applyAsync` 不会马上调用传入的函数，而且也不会立即触发一个 digest。相反，`$applyAsync` 会推迟一段比较短的时间之后再执行这两项工作。

`$applyAsync` 最初的设计动机是用于处理 HTTP 响应：当 `$http` 服务收到一个响应之后，就会调用一些响应处理函数并且启动一个 digest。这意味着每次 HTTP 响应到来时都要运行一个 digest。这样的话，如果应用的 HTTP 通信比较频繁或者它的 digest 循环需要花费较多资源时（很多应用启动时都可能会遇到这种情况），就会产生性能问题。现在，`$http` 就可以配置为使用 `$applyAsync` 的模式，对于到达时间非常接近的 HTTP 应答会被合并到一个 digest 中处理。然而， `$applyAsync` 并没有绑定到 `$http` 服务中去，你可以把它应用到任何可以从“合并到一个 digest 进行处理“的模式中获益的情况中去。

在关于 `$applyAsync` 的第一个测试用例中，我们希望调用这个函数后不会立马发生改变，在过了 50 毫秒以后才会产生变化：

_test/scope_spec.js_

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

到目前为止，我们还没有发现 `$applyAsync` 跟 `$evalAsync` 有什么区别，但我们只要在 listner 函数中调用一下 `$applyAsync` 就能发现端倪。如果我们在这里调用 `$evalAsync`，那么这个函数仍然会在同一个 digest 中被调用。但 `$applyAsync` 总是会延迟函数的调用：

_test/scope_spec.js_

```js
it('never executes $applyAsynced function in the same cycle', function(done) {
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

作为开发 `$applyAsync` 的第一步，我们需要在 Scope 构造函数中添加另一个队列。这个队列就用于存放我们使用 `$applyAsync` 设定的异步任务：

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

每调用一次 `$applyAsync`，我们就会把一个函数添加到这个队列中来。稍后，这个函数就会在当前作用域语境下对传入的表达式进行运算，这跟 `$apply` 是一样的：

_src/scope.js_

```js
Scope.prototype.$applyAsync = function(expr) {
  var self = this;
  self.$$applyAsyncQueue.push(function() {
    self.$eval(expr);
  });
};
```

这里我们还需要设定用于调用这些函数的延时任务。我们可以直接用 `setTimeout` 并传入值为 0 的 dalay 参数来实现延时。在这个定时器函数中，我们会用 `$apply` 来触发一个函数，这个函数将会遍历异步任务队列，逐一对队列中存储的函数进行调用：

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

> 注意，我们不会对队列中的每一个元素都调用一次 `$apply`。我们只需要在循环以外调用一次 `$apply` 就可以了，毕竟我们只需要启动一次 digest。

正如我们上面说到的，`$applyAsync`的核心是对一些高频操作进行优化，让这些操作带来的变化用一次 digest 就能进行处理。我们现在还没有真正实现这个目标。目前的情况是，每次调用 `$applyAsync` 都会设定一个启动 digest 的延时任务，我们在 watch 函数中添加一个计数器就能发现问题了：

_test/scope_spec.js_

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

我们希望计数器的数字能到 2（因为在第一次 digest 中 watch 会被执行两次），但不能大于这个数字。

我们要做的就是记录用于执行异步任务队列的 `timeout` 定时器是不是已经设定好了。我们会在把这个信息保存在私有的作用域属性 `$$applyAsyncId` 中：

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

在设定延时任务之前，我们需要先检查一下这个属性，并在设定好之后和任务完成之后对这个属性进行更新：

_src/scope.js_

```js
Scope.prototype.$applyAsync = function(expr) {
  var self = this;
  self.$$applyAsyncQueue.push(function() {
    self.$eval(expr);
  });
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

关于 `$applyAsync` 的另一个事实是，如果在 timeout 定时器触发之前因为其他某些原因已经启动了一个 digest，那定时器中的 digest 就无需启动了。在这种情况下，当前在运行 digest 就能把异步任务执行完，而 `$applyAsync` 的定时器应该被销毁了：

_test/scope_spec.js_

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

在这里，我们验证 `$applyAsync` 设定的任务是否都会在调用 `$digest` 之后就会马上执行，这样后面就没剩下需要执行的任务了。

下面，我们先把 `$applyAsync` 中用于遍历执行异步任务队列的代码抽取成一个内部函数，这样我们就可以在多处地方使用了：

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

> LoDash 的 `_.bind` 函数就相当于 ECMAScript 5 的 `Function.prototype.bind`，是用于确保函数的 this 指向一个已知值的。

现在，我们就可以在 `$digest` 函数中直接调用这个函数了，并判断如果当前已经有定时器处于等待触发状态，我们就取消这个定时器并立即开始遍历异步任务队列：

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

这就是 `$applyAsync` 的全部内容了。它实际上就是在 `$apply` 的基础进行了一些小优化，在一些需要调用 `$apply` 但会在短时间内多次调用的情况能派上用场。
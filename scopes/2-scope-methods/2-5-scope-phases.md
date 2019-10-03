### 作用域阶段
#### Scope Phases

`$evalAsync`还可以在检测到当前没有 digest 运行时定时调用 `$digest` 方法。那意味着，无论你在什么时候使 `$evalAsync` 来延迟执行一个函数，你都可以保证这个函数会在“不久后”被调用，而不需要等待其他东西来启动一个 digest。

> 虽然 `$evalAsync` 会定时启动一个 `$digest`，但对于异步执行代码，我们更推荐使用 `$applyAsync`，下一节中会讲到。

要完成这个功能，首先我们需要让 `$evalAsync` 可以通过某种途径检查到当前是否有 digest 正在运行，因为如果当前已经有 digest 在运行的话，我们就不需要再特地启动一个 digest 了。为此，Angular 作用域加入了 _作用域阶段_ 这个概念，实际上它就是作用域对象上的一个字符串类型的属性，在这个属性里面存储了当前 digest 运行状态的信息。

在单元测试中，我们假设作用域对象上有一个属性叫 `$$phase`，它的值在 digest 运行过程中会是 `“$digest”`，而在 apply 一个函数的调用时会是 `“$apply”`，其他时间就是 `null`。下面我们会在 `describe('$digest')` 的测试集中加入这个单元测试：

_test/scope_spec.js_

```js
it('has a $$phase field whose value is the current digest phase', function() {
  scope.aValue = [1, 2, 3];
  scope.phaseInWatchFunction = undefined;
  scope.phaseInListenerFunction = undefined;
  scope.phaseInApplyFunction = undefined;

  scope.$watch(
    function(scope) {
      scope.phaseInWatchFunction = scope.$$phase;
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {
      scope.phaseInListenerFunction = scope.$$phase;
    }
  );

  scope.$apply(function(scope) {
    scope.phaseInApplyFunction = scope.$$phase;
  });

  expect(scope.phaseInWatchFunction).toBe('$digest');
  expect(scope.phaseInListenerFunction).toBe('$digest');
  expect(scope.phaseInApplyFunction).toBe('$apply');

});
```

> 注意，这里我们无需显式地调用 `$digest` 方法来启动对作用域的 digest，因为调用 `$apply` 就包含了这一步了。

在 `Scope` 构造函数中，我们会加入 `$$phase` 这个实例属性，并将它的初始值设为 `null`：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
  this.$$lastDirtyWatch = null;
  this.$$asyncQueue = [];
  this.$$phase = null;
}
```

下面，我们会定义两个用于控制作用域周期的函数：一个用于写入值，另一个用于清空当前值。同时，我们也会加入一个额外的检查，如果当前作用域阶段已经存在有效值的时候，就不能再尝试写入：

_src/scope.js_

```js
Scope.prototype.$beginPhase = function(phase) {
  if (this.$$phase) {
    throw this.$$phase + ' already in progress.';
  }
  this.$$phase = phase;
};

Scope.prototype.$clearPhase = function() {
  this.$$phase = null;
};
```

在 `$digest` 方法中，在开始 digest 循环之前我们就把作用域阶段设置为 `“$digest”`：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  this.$beginPhase(‘$digest’);
  // do {
  //   while (this.$$asyncQueue.length) {
  //     var asyncTask = this.$$asyncQueue.shift();
  //     asyncTask.scope.$eval(asyncTask.expression);
  //   }
    // dirty = this.$$digestOnce();
    // if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
      this.$clearPhase();
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty || this.$$asyncQueue.length);
  this.$clearPhase();
};
```

在 `$apply` 方法中，我们也要把作用域阶段设置为 `$apply`：

_src/scope.js_

```js
Scope.prototype.$apply = function(expr) {
  try {
    this.$beginPhase(‘$apply’);
    // return this.$eval(expr);
  } finally {
    this.$clearPhase();
    // this.$digest();
  }
};
```

最后，我们再把定时执行 `$digest` 的功能加入到 `$evalAsync` 中。首先，我们还是把这个需求定义为一个单元测试，并把它放到 `describe('$evalAsync')` 测试集中：

_test/scope_spec.js_

```js
it('schedules a digest in $evalAsync', function(done) {
  scope.aValue = 'abc';
  scope.counter = 0;

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  
  scope.$evalAsync(function(scope) {});
  
  expect(scope.counter).toBe(0);
  setTimeout(function() {
    expect(scope.counter).toBe(1);
    done();
  }, 50);
});
```

我们在 `$evalAsync` 调用后过去一段时间，才去检查 digest 是否运行了，而不是在调用后马上检查。具体来说，这里的“过去一段时间”就是使用 50 毫秒的 timeout 定时器。要在 Jasmine 中使用 `setTimeout`，我们要使用它对异步测试的支持方式：测试案例对应的函数可以传入一个名为 `done` 的回调函数参数，在调用这个回调函数之后这个单元测试进程才会结束。我们会在 timeout 定时器的最后调用这个回调函数。

`$evalAsync` 函数现在就可以检查作用域的当前状态，如果作用域状态为空，我们就设定一个延迟执行的 digest 任务。

_src/scope.js_

```js
Scope.prototype.$evalAsync = function(expr) {
  var self = this;
  if (!self.$$phase && !self.$$asyncQueue.length) {
    setTimeout(function() {
      if (self.$$asyncQueue.length) {
        self.$digest();
      }
    }, 0);
  }
  // self.$$asyncQueue.push({ scope: self, expression: expr });
};
```

注意，我们会在两处地方检查异步任务队列的长度：

- 在调用 `setTimeout` 之前，我们要保证这个队列是空的。那是因为我们不希望 `setTimeout` 的调用次数超出我们的预期。如果异步任务队列还没清空，说明我们已经设置了一个 timeout 定时任务，这个定时任务中启动的 digest 就会把这些异步任务队列都执行掉。
- 在 `setTimeout` 函数中，我们需要保证这个队列是非空的。异步任务队列可能会在 timeout 定时任务执行之前由于其他原因已经全部执行掉了，这时如果没事可干，就没必要再启动一个 digest 了。

有了以上的设置，我们就能确信，无论何时在何处调用 `$evalAsync` ，都会有一个 digest 在调用之后的一段时间启动起来。

如果在调用 `$evalAsync` 的时候已经有一个 digest 在运行，那我们传入的函数就会在这个 digest 中得到处理。如果调用时没有 digest 在运行，`$evalAsync` 就会自动启动一个 digest。我们会使用 `setTimeout` 来稍微推迟这个 digest 的开始时间。通过这种方式，`$evalAsync` 的调用者就可以保证无论当前 digest 处于什么阶段，调用这个函数后就会立即返回，而无需等待对表达式的同步（synchronously）运算。
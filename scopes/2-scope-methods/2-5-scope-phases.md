### 作用域阶段

#### Scope Phases

`$evalAsync` 在检测到当前没有 digest 运行时还会定时启动一个 `$digest` 周期。这意味着，无论你在什么时候利用 `$evalAsync` 来延迟执行一个函数，都可以肯定这个函数会在“不久后”被调用，而不需要依赖其他事件触发 digest。

> 注意，虽然 `$evalAsync` 也可以异步启动一个 `$digest`，但我们还是推荐你使用 `$applyAsync` 来完成这个需求，我们会下节介绍这个方法。

要完成这个需求，我们先得让 `$evalAsync` 可以通过某种途径检查到当前是否有 digest 正在运行，这是因为如果当前已经有 digest 在运行，我们就没有必要再启动一个 digest 了。为此，Angular 实现了一个叫 _作用域阶段_（phase）的东西，它就是作用域对象上的一个字符串属性而已，用于标识当前发生的状态或事件。

在单元测试中，我们假定作用域对象上有一个叫 `$$phase` 的属性，在 digest 运行过程中，它的值会是 `“$digest”`，而在使用 apply 调用一个函数的时会变成 `“$apply”`，在其他所有时间都会是 `null`。下面我们会在 `describe('$digest')` 测试集合中加入这个单元测试：

_test/scope\_spec.js_

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

> 注意，这里我们无需再调用 `$digest` 方法来启动 digest，因为调用 `$apply` 时就会帮我们顺带启动一个。

在 `Scope` 构造函数中，我们会加入 `$$phase` 这个实例属性，并把它初始化为 `null`：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
  this.$$lastDirtyWatch = null;
  this.$$asyncQueue = [];
  this.$$phase = null;
}
```

下面，我们会定义两个用于控制 phase 的函数：一个用于设置，另一个用于清空。同时，我们也会加入一个额外的检查，确保 phase 不会在处于激活状态时被更改：

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

在 `$digest` 中，我们会在 digest 循环开始之前把 phase 设置为 `“$digest”`：

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

调用 `$apply` 方法时，我们要把 phase 设置为 `$apply`：

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

最后我们再把定时执行 `$digest` 的功能加入到 `$evalAsync` 中。但在此之前还是先把这个需求定义为一个单元测试放到 `describe('$evalAsync')` 测试集合中：

_test/scope\_spec.js_

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

我们会在 `$evalAsync` 调用完一段时间后才会去检查 digest 是否有运行，而不是在调用后马上检查。这里说的“一段时间”实际上就是 timeout 定时器设定的 50 毫秒。要在 Jasmine 中使用 `setTimeout`，我们就要用到它对异步测试的支持：测试用例对应的函数可以传入一个名为 `done` 的回调函数参数，只有调用回调函数才会真正结束这个单元测试进程。我们会在 timeout 函数的最后调用这个回调函数。

现在 `$evalAsync` 函数可以检查作用域的当前状态，如果作用域状态为空，就会设定一个延迟执行的 digest 任务。

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

注意，我们需要在两处地方检查异步任务队列的长度：

* 在调用 `setTimeout` 之前，我们要保证这个队列是空的。那是因为我们不希望 `setTimeout` 的调用次数超出我们的预期。如果异步任务队列还没清空，说明现在已经有一个 timeout 定时任务在运行了，这个定时任务中的 digest 会把这些异步任务队列都执行掉，无需再开一个定时器。
* 在 `setTimeout` 函数中，我们需要保证这个队列是非空的。异步任务队列可能会在 timeout 定时任务执行之前由于某些原因（译者注）已经被全部执行掉了，定时任务无事可干，也就没有必要再启动一个 digest 了。

> 译者注：比如有个正在执行的 digest 循环已经把异步任务都完成了

开发到这里，我们就能确保无论在何时何处调用 `$evalAsync` ，都会有一个 digest 在稍后被启动。

如果在调用 `$evalAsync` 时 digest 正在运行，那我们传入的函数就会在这个 digest 中得到处理。相反，如果调用时没有 digest 在运行，`$evalAsync` 就会帮我们启动一个 digest。我们使用 `setTimeout` 来推迟这个 digest 的开始时间。这样无论当前 digest 处于什么阶段，`$evalAsync` 的调用者就可以确保函数总是立即返回，无需等待对表达式的同步（synchronously）运算。


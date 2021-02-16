### 在 digest 周期结束时运行代码—— $$postDigest

#### Running Code After A Digest - $$postDigest

有一个方法可以让我们可以在 digest 周期结束后再执行代码，那就是 `$$postDigest`。

这个方法以两个美元符号作为前缀，暗示它实际上是 Angular 的内部方法，不开放给应用开发者使用。但既然存在这个方法，那我们也把它实现一下。

像 `$evalAsync` 和 `$applyAsync` 一样，`$$postDigest` 也能延迟“一段时间”再运行传入的函数。具体来说，这个函数会在下一次 digest 完成时被调用。跟 `$evalAsync` 类似，使用 `$$postDigest` 延时执行的函数也只会被执行一次。但它跟 `$evalAsync` 和 `$applyAsync` 不同的是，`$$postDigest` 设定延时函数时，并不会设定延时启动 digest 任务，因此这个函数会延迟到下一个被（某种原因）触发的 digest 运行后才会被执行。下面我们使用单元测试来对这些需求进行明确定义：

_test/scope\_spec.js_

```js
describe('$postDigest', function() {
  var scope;

  beforeEach(function() {
    scope = new Scope();
  });

  it('runs after each digest', function() {
    scope.counter = 0;
    scope.$$postDigest(function() {
      scope.counter++;
    });

    expect(scope.counter).toBe(0);
    scope.$digest();

    expect(scope.counter).toBe(1);
    scope.$digest();

    expect(scope.counter).toBe(1);
  });
});
```

顾名思义，`$$postDigest` 函数是在 digest 运行之后才执行的，所以如果你在 `$$postDigest` 中对作用域中的数据进行修改时，这些变化并不会马上被脏值检测系统捕获到。如果你希望捕捉到这些变化，你需要再次调用 `$digest` 或 `$apply`：

_test/scope\_spec.js_

```js
it('does not include $$postDigest in the digest', function() {
  scope.aValue = 'original value';

  scope.$$postDigest(function() {
    scope.aValue = 'changed value';
  });
  scope.$watch(
    function(scope) {
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {
      scope.watchedValue = newValue;
    }
  );

  scope.$digest();
  expect(scope.watchedValue).toBe('original value');

  scope.$digest();
  expect(scope.watchedValue).toBe('changed value');
});
```

实现 `$$postDigest` 的第一步是在 `Scope` 构造函数再创建一个数组类型的实例属性：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  // this.$$applyAsyncQueue = [];
  // this.$$applyAsyncId = null;
  this.$$postDigestQueue = [];
  // this.$$phase = null;
}
```

接着就要实现 `$$postDigest` 本身了。这个函数要做的仅仅是把传入的函数添加到上面新增的数组中就可以了：

_src/scope.js_

```js
Scope.prototype.$$postDigest = function(fn) {
  this.$$postDigestQueue.push(fn);
};
```

最后，在 `$digest` 函数中，我们需要在 digest 运行结束时对这个队列进行遍历，逐个调用里面的函数：

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

  // do {
  //   while (this.$$asyncQueue.length) {
  //     var asyncTask = this.$$asyncQueue.shift();
  //     asyncTask.scope.$eval(asyncTask.expression);
  //   }
  //   dirty = this.$$digestOnce();
  //   if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
  //     this.$clearPhase();
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty || this.$$asyncQueue.length);
  // this.$clearPhase();

  while (this.$$postDigestQueue.length) {
    this.$$postDigestQueue.shift()();
  }
};
```

利用 `Array.shift()` 可以不断地从数组中取出排列在最前面的函数进行调用，一直持续到数组被清空。我们无需对 `$$postDigest` 传入任何参数。


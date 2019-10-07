### 在 digest 之后运行代码—— $$postDigest
Running Code After A Digest - $$postDigest

有一种方法让我们可以在 digest 周期结束后执行代码，就是通过设置 `$$postDigest` 函数来实现。

这个函数名中的两个美元符号表明这个函数只用于 Angular 内部，而不是给应用开发者使用的。但既然存在这个函数，我们也把它实现出来。

像 `$evalAsync` 和 `$applyAsync` 一样，`$$postDigest` 会让传入的函数在“稍后一段时间”再运行。具体来说，这个函数会在接下来的一次 digest 完成时才会被调用。类似 `$evalAsync`，使用 `$$postDigest` 延时执行的函数只会被执行一次。但跟 `$evalAsync` 和 `$applyAsync` 不同的是，设置 `$$postDigest` 延时函数后不会同时设定一个延时启动 digest 的定时任务，因此这个函数会延迟到接下来的、因某种原因触发的 digest 启动之后才会执行。下面的单元测试就会对这些需求进行明确定义：

_test/scope_spec.js_

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

顾名思义，`$$postDigest` 函数是 digest 之后才运行的，因此如果你在 `$$postDigest` 中对作用域中的数据进行修改时，这些变化并不会马上被脏值检测系统捕获到。如果你确实希望这些变化能被捕获到，你就需要手动调用 `$digest` 或 `$apply`：

_test/scope_spec.js_

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

开发 `$$postDigest` 的第一步是在 `Scope` 构造函数再初始化一个数组类型的实例属性：

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

下一步就是实现 `$$postDigest` 本身了。这个函数要做的仅仅是把传入的函数添加到上面新增的队列中：

_src/scope.js_

```js
Scope.prototype.$$postDigest = function(fn) {
  this.$$postDigestQueue.push(fn);
};
```

最后，在 `$digest` 函数内部，我们需要在 digest 运行结束时对这个队列进行遍历，并逐个调用里面的函数：

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

我们利用 `Array.shift()` 从数组中提取最前面的函数并立即调用，同时可以不断缩小队列长度直到队列被清空。我们无需对 `$$postDigest` 传入任何参数。
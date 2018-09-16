### 解决 Deferred（Resolving A Deferred）

现在我们已经知道怎么创建 Deferred 和 Promise 了，下面将介绍他们在异步调用中的真正用处。

当我们创建了 Deferred 和 Promise 后，我们就可以为 Promise 添加一个回调函数。然后，当产生结果值时，我们就可以通过 Deferred 把结果值传递到 Promise。在之后的某个时刻，之前注册在 Promise 的回调函数就会以该值作为参数进行调用。我们可以使用 Jasmine 的 spy 函数来测试这一流程：

```js
it('can resolve a promise', function(done) {
  var deferred = $q.defer();
  var promise = deferred.promise;

  var promiseSpy = jasmine.createSpy();
  promise.then(promiseSpy);

  deferred.resolve('a-ok');

  setTimeout(function() {
    expect(promiseSpy).toHaveBeenCalledWith('a-ok');
    done();
  }, 1);
});
```

上文提到的“在之后的某个时刻”还是比较模糊，随着之后的解析，我们会对其概念更清晰。现在，我们先让测试用例通过了再说。

Promise 会把它的内部状态存储到一个叫 $$state 的内部对象变量中：

```js
function Promise() {
  this.$$state = {};
}
```

当 Promise 的 then 方法被调用后，then 的参数，也就是回调函数（callback）会被存储到 $$state 的 pending 属性中：

```js
function Promise() {
  this.$$state = {};
}
Promise.prototype.then = function(onFulflled) {
  this.$$state.pending = onFulflled;
};
```

然后，当对应的 Deferred 获取到了计算结果，我们会调用 Promise 中保存的回调函数：

```js
function Deferred() {
  this.promise = new Promise();
}
Deferred.prototype.resolve = function(value) {
  this.promise.$$state.pending(value);
};
```

这样我们就可以满足测试用例的要求了，但对于我们真正的应用需要来，这还是太过于简单了。比如，当前这个版本对于处理流程有着严格的要求。对于结果值已经计算出来，我们应该要支持在 Deferred 运算已结束的情况下注册回调函数，该回调函数也要可以被调用：

```js
it('works when resolved before promise listener', function(done) {
  var d = $q.defer();
  d.resolve(42);
  var promiseSpy = jasmine.createSpy();
  d.promise.then(promiseSpy);
  setTimeout(function() {
    expect(promiseSpy).toHaveBeenCalledWith(42);
    done();
  }, 0);
});
```

另外，正如之前提到的，我们也要保证 Promise 回调函数在结果值已经计算得出时并不会马上被调用。这也是我们在第一个测试用例中使用 setTimeout 的原因。

```js
it('does not resolve promise immediately', function() {
  var d = $q.defer();
  var promiseSpy = jasmine.createSpy();
  d.promise.then(promiseSpy);
  d.resolve(42);
  expect(promiseSpy).not.toHaveBeenCalled();
});
```

那么到底什么时候会调用 Promise 回调函数呢？这时就体现出 digest 循环的用处。Promise 回调函数将会在 Deferred 被 resolve 后的下一个 digest 时被调用。为了测试，我们需要把 Scope 引入到我们的测试文件中：

```js
var $q, $rootScope;
beforeEach(function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  $q = injector.get('$q');
  $rootScope = injector.get('$rootScope');
});
```

现在，我们可以测试一下到底 Promise 回调函数是否会在 digest 期间被调用：

```js
it('resolves promise at next digest', function() {
  var d = $q.defer();

  var promiseSpy = jasmine.createSpy();
  d.promise.then(promiseSpy);
  
  d.resolve(42);
  $rootScope.$apply();
  
  expect(promiseSpy).toHaveBeenCalledWith(42);
});
```

我们将会把计算出来的结果值保存起来，并且利用一个帮助函数来达到延迟调用的效果：

```js
Deferred.prototype.resolve = function(value) {
  this.promise.$$state.value = value;
  scheduleProcessQueue(this.promise.$$state);
};
```

这个帮助函数实质上利用了根 Scope 的 $evalAsync。让我们回忆下， $evalAsync 可以让我们在下一次 digest 时调用函数，也可以在回调函数没有执行计划时，使用 setTimeout 定时调用。同时 $evalAsync 的回调函数也会调用另一个帮助函数：

```js
function scheduleProcessQueue(state) {
  $rootScope.$evalAsync(function() {
    processQueue(state);
  });
}
```

为了使用 $evalAsync，我们需要在 $QProvider 里面饮入 $rootScope，所以让我们会在 $QProvider 的 $get 方法中注入 $rootScope：

```js
this.$get = ['$rootScope', function($rootScope) {
  // ...
}];
```

最后我们需要在帮助函数 processQueue 中加入实际的调用代码：

```js
function processQueue(state) {
  state.pending(state.value);
}
```

> 大部分 Promise 库（包括 Q），当 Deferred resolve 时，它的回调函数也不是立马就被调用的。当然，它们没有 AngularJS 的 digest 循环机制，只能使用像 setTimeout 之类的东西。

AngularJS 在 digest 期间传递 resolve 结果，这种做法可能会带来性能上的优化。由于 Promise 回调函数会被异步调用，如果这时候结果值已经准备就绪，这个回调函数就会在同一个 JavaScript 事件循环中被调用。这意味着，我们无需把函数调用的控制权交还给浏览器，毕竟在交换控制权后浏览器是有机会去做一些别的事，这在使用 setTimeout 的 promise 库中很常见。
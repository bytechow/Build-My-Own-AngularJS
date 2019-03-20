### 注册多个 Promise 回调（Registering Multiple Promise Callbacks）

虽然一个 Deferred 只可以对应一个 Promise，一个 Promise 可以拥有多个回调。现在的代码还未对此进行支持：

```js
it('may have multiple callbacks', function() {
  var d = $q.defer();

  var firstSpy = jasmine.createSpy();
  var secondSpy = jasmine.createSpy();
  d.promise.then(firstSpy);
  d.promise.then(secondSpy);

  d.resolve(42);
  $rootScope.$apply();

  expect(firstSpy).toHaveBeenCalledWith(42);
  expect(secondSpy).toHaveBeenCalledWith(42);
});
```

要满足该特性，我们需要在 digest 运行且 Promise 已经被 resolve 的情况下，对所有与该 Promise 对应的还没有被调用的 callback 进行调用。但正如我们刚才提到的，每个回调应该只会被调用一次：

```js
it('invokes each callback once', function() {
  var d = $q.defer();

  var firstSpy = jasmine.createSpy();
  var secondSpy = jasmine.createSpy();

  d.promise.then(firstSpy);
  d.resolve(42);
  $rootScope.$apply();
  expect(firstSpy.calls.count()).toBe(1);
  expect(secondSpy.calls.count()).toBe(0);

  d.promise.then(secondSpy);
  expect(firstSpy.calls.count()).toBe(1);
  expect(secondSpy.calls.count()).toBe(0);

  $rootScope.$apply();
  expect(firstSpy.calls.count()).toBe(1);
  expect(secondSpy.calls.count()).toBe(1);
});
```

现在，我们还仅支持注册一个 callback ，接下来，我们要支持注册多个 callback，所以我们会采用数组结构：

```js
Promise.prototype.then = function(onFulflled) {
  this.$$state.pending = this.$$state.pending || [];
  this.$$state.pending.push(onFulflled);
  if (this.$$state.status > 0) {
    scheduleProcessQueue(this.$$state);
  }
};
```

然后，我们会依次调用它们：

```js
function processQueue(state) {
  _.forEach(state.pending, function(onFulflled) {
    onFulflled(state.value);
  });
}
```

为了能使用\_.forEach，我们需要引入 lodash：

```js
'use strict';
var _ = require('lodash');
```

另外，为了确保每个回调只会被调用一次，我们需要在调用时先把 pending 的 callback 都清空。也就是说，在每个 digest 中，pending 的 callback 都会被清空。如果之后注册了其它回调，pending 会被重新初始化（为空数组）：

```js
function processQueue(state) {
  var pending = state.pending;
  state.pending = undefned;
  _.forEach(pending, function(onFulflled) {
    onFulflled(state.value);
  });
}
```

> 这里的程序执行顺序是需要注意的。If some of the callbacks happen to register more callbacks, we don’t want them to be cleared just yet.




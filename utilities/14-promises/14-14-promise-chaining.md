### Promise 链（promise-chaining）

上面我们已经了解到 Deferred 和 Promise 是怎么配合运作的，对于执行一个异步任务来说已经足够了，但 Promise 的价值在于它支持依次多个异步任务，形成一个工作流。这个工作流也就是我们需要实现的 Promise 链。

Promise 链的最基本用法就是，用 then 依次链接各个流程的回调函数。每个回调会接收从上一个回调的返回值作为参数：

```js
it('allows chaining handlers', function() {
  var d = $q.defer();

  var fulflledSpy = jasmine.createSpy();
  d.promise.then(function(result) {
    return result + 1;
  }).then(function(result) {
    return result * 2;
  }).then(fulflledSpy);

  d.resolve(20);
  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith(42);
});
```

这里的关键是 then 方法每次调用都会生成一个新的 Promise 对象，方便下个工作流程绑定回调。这里，理解到每次调用都会生成一个新的 Promise 很关键。这样的话，基于源 Promise 的其他回调不会受到影响：

```js
it('does not modify original resolution in chains', function() {
  var d = $q.defer();

  var fulflledSpy = jasmine.createSpy();
  d.promise.then(function(result) {
    return result + 1;
  }).then(function(result) {
    return result * 2;
  });
  d.promise.then(fulflledSpy);

  d.resolve(20);
  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith(20);
});
```

我们需要在 then 方法里面创建一个新的 Deferred——它代表在本次工作流程中 onFulfilled 回调的计算结果。之后可以 then 方法的最后直接返回它的 Promise：

```js
Promise.prototype.then = function(onFulflled, onRejected) {
  var result = new Deferred();
  this.$$state.pending = this.$$state.pending || [];
  this.$$state.pending.push([null, onFulflled, onRejected]);
  if (this.$$state.status > 0) {
    scheduleProcessQueue(this.$$state);
  }
  return result.promise;
};
```

我们还得把 Defferred  对应的计算结果放到 onFulfilled 回调被调用的地方才行，这样才能实现传递。我们会把这个结算结果作为 $$state.pending 第一个参数：

```js
Promise.prototype.then = function(onFulflled, onRejected) {
  var result = new Deferred();
  this.$$state.pending = this.$$state.pending || [];
  this.$$state.pending.push([result, onFulflled, onRejected]);
  if (this.$$state.status > 0) {
    scheduleProcessQueue(this.$$state);
  }
  return result.promise;
};
```




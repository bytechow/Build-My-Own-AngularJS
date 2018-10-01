拒绝 Deffered，捕捉拒绝决议

正如我们在本章开始时提到的，回调函数处理异步时有一个弊端，就是缺乏一种统一的错误处理机制。我们可能会选择某一个的回调函数参数传递错误，但大部分库或应用对其的处理都大不相同。

Promise 通过内建的错误处理机制来处理错误。这一套解决方案由两部分组成，第一部分是 Promise 提供了一种显式地标识程序出现错误的方式，我们称之为拒绝决议。第二部分是出错时可以通知到对应的程序进行处理，我们会允许 then 提供第二个参数，这个参数是一个回调函数，当 Promise 被拒绝时就会被调用：

```js
it('can reject a deferred', function() {
  var d = $q.defer();

  var fulfllSpy = jasmine.createSpy();
  var rejectSpy = jasmine.createSpy();
  d.promise.then(fulfllSpy, rejectSpy);

  d.reject('fail');
  $rootScope.$apply();

  expect(fulfllSpy).not.toHaveBeenCalled();
  expect(rejectSpy).toHaveBeenCalledWith('fail');
});
```

一个 Promise 只能被拒绝一次：

```js
it('can reject just once', function() {
  var d = $q.defer();

  var rejectSpy = jasmine.createSpy();
  d.promise.then(null, rejectSpy);

  d.reject('fail');
  $rootScope.$apply();
  expect(rejectSpy.calls.count()).toBe(1);

  d.reject('fail again');
  $rootScope.$apply();
  expect(rejectSpy.calls.count()).toBe(1);
});
```

当然，我们也不能将一个已被拒绝的决议变成完成（与之相反的情况同样不允许）。也就是说，Promise 最多只能有一个结果，要么是完成，要么是拒绝：

```js
it('cannot fulfll a promise once rejected', function() {
  var d = $q.defer();

  var fulfllSpy = jasmine.createSpy();
  var rejectSpy = jasmine.createSpy();
  d.promise.then(fulfllSpy, rejectSpy);

  d.reject('fail');
  $rootScope.$apply();

  d.resolve('success');
  $rootScope.$apply();

  expect(fulfllSpy).not.toHaveBeenCalled();
});
```

Deffered 的 reject 方法跟 resolve 方法类似。唯一的不同就是，如果是 Promise  被 reject，我们会将 Promise 的内部状态设置为2，而不是1：

```js
Deferred.prototype.reject = function(reason) {
  if (this.promise.$$state.status) {
    return;
  }
  this.promise.$$state.value = reason;
  this.promise.$$state.status = 2;
  scheduleProcessQueue(this.promise.$$state);
};
```

Promise 的 then 方法现在需要支持接收两个回调，一个是 Promise 完成时的回调，另一个是失败时的回调。pending 属性现在也要变成一个存放回调的数组了。我们会把完成时回调放在数组下标为1的位置，而把拒绝时回调放在下标为2的位置。这样我们的决议状态就可以与数组下标保持一致（PS: 这样安排数组位置还有其他考量，下文会介绍）：

```js
Promise.prototype.then = function(onFulflled, onRejected) {
  this.$$state.pending = this.$$state.pending || [];
  this.$$state.pending.push([null, onFulflled, onRejected]);
  if (this.$$state.status > 0) {
    scheduleProcessQueue(this.$$state);
  }
};
```

在 processQueue 方法，我们可以通过获取当前 Promise 的决议状态来决定使用哪个回调，并不需要额外工作来进行区分：

```js
function processQueue(state) {
  var pending = state.pending;
  state.pending = undefned;
  _.forEach(pending, function(handlers) {
    var fn = handlers[state.status];
    fn(state.value);
  });
}
```

当前的实现还有一个问题，就是我们目前默认每一个 then 方法调用都会给出完成时回调和拒绝时回调。事实上，我们应该允许忽略其中一个回调（技术上两个回调都可以忽略）：

```js
it('does not require a failure handler each time', function() {
  var d = $q.defer();

  var fulfllSpy = jasmine.createSpy();
  var rejectSpy = jasmine.createSpy();
  d.promise.then(fulfllSpy);
  d.promise.then(null, rejectSpy);

  d.reject('fail');
  $rootScope.$apply();

  expect(rejectSpy).toHaveBeenCalledWith('fail');
});

it('does not require a success handler each time', function() {
  var d = $q.defer();

  var fulfllSpy = jasmine.createSpy();
  var rejectSpy = jasmine.createSpy();

  d.promise.then(fulfllSpy);
  d.promise.then(null, rejectSpy);

  d.resolve('ok');
  $rootScope.$apply();

  expect(fulfllSpy).toHaveBeenCalledWith('ok');
});
```

要修复这个问题，我们会对传入的回调函数的类型进行检测：

```js
function processQueue(state) {
  var pending = state.pending;
  state.pending = undefned;
  _.forEach(pending, function(handlers) {
    var fn = handlers[state.status];
    if (_.isFunction(fn)) {
      fn(state.value);
    }
  });
}
```

对于 Promise.then\(null, callback\) 这种只有错误回调的监听器，我们可以使用一个便捷方法：

```js
it('can register rejection handler with catch', function() {
  var d = $q.defer();

  var rejectSpy = jasmine.createSpy();
  d.promise.catch(rejectSpy);
  d.reject('fail');
  $rootScope.$apply();

  expect(rejectSpy).toHaveBeenCalled();
});
```

实际上 catch 方法在底层也只是调用 then，并没有什么特别之处：

```js
Promise.prototype.catch = function(onRejected) {
  return this.then(null, onRejected);
};
```




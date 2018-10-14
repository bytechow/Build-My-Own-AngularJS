###  进度通知（Notifying Progress）

当我们在进行异步操作时，有时可能会需要知道当前的完成进度。虽然还没有完成 Promise，但我们希望发送一些信息让人知道现在正在发生什么。

Angular 的 $q 服务内置了这种进度通知功能——也就是 Deferred 的原型方法 notify。你可以通过它传递一些信息到对应的监听者。具体来说，你可以在 then 方法 的第三个参数中传入一个回调，这就是 notify 服务的监听者：

```js
it('can report progress', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  d.promise.then(null, null, progressSpy);

  d.notify('working...');
  $rootScope.$apply();

  expect(progressSpy).toHaveBeenCalledWith('working...');
});
```

notify 方法与 resolve、reject 这些方法的不同之处在于，我们可以调用 notify 多次，对应的回调也会被调用多次。正如我们之前所说的，一个 Promise 不能被多次 resolve（或reject）。

```js
it('can report progress many times', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  d.promise.then(null, null, progressSpy);

  d.notify('40%');
  $rootScope.$apply();

  d.notify('80%');
  d.notify('100%');
  $rootScope.$apply();
  
  expect(progressSpy.calls.count()).toBe(3);
});
```

要实现这一个功能，首先我们需要保存 notify 方法里的回调参数。我们会把它作为 pending array 中的第四个参数：

```js
Promise.prototype.then = function(onFulflled, onRejected, onProgress) {
  var result = new Deferred();
  this.$$state.pending = this.$$state.pending || [];
  this.$$state.pending.push([result, onFulflled, onRejected, onProgress]);
  if (this.$$state.status > 0) {
    scheduleProcessQueue(this.$$state);
  }
  return result.promise;
};
```

然后我们就可以着手实现 notify 方法了。它会遍历找到所有的 notify 回调，并以此调用。为了遵循 Promise 的异步原则，我们并不会立即调用 notify 回调，而是会延迟一点点，因为我们借助了 $rootScope.$evalAsync 方法：

```js
Deferred.prototype.notify = function(progress) {
  var pending = this.promise.$$state.pending;
  if (pending && pending.length) {
    $rootScope.$evalAsync(function() {
      _.forEach(pending, function(handlers) {
        var progressBack = handlers[3];
        if (_.isFunction(progressBack)) {
          progressBack(progress);
        }
      });
    });
  }
};
```

让 notify 回调异步进行的特性在一种情况下特别有用。这种请况就是当 Promise 已经 resolve 时，notify 回调就不会再被调用了：

```js
it('does not notify progress after being resolved', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  d.promise.then(null, null, progressSpy);

  d.resolve('ok');
  d.notify('working...');
  $rootScope.$apply();

  expect(progressSpy).not.toHaveBeenCalled();
});
```

对于 rejection 也是一样的：

```js
it('does not notify progress after being rejected', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  d.promise.then(null, null, progressSpy);

  d.reject('fail');
  d.notify('working...');
  $rootScope.$apply();
  
  expect(progressSpy).not.toHaveBeenCalled();
});
```

我们可以通过在 notify 原型方法中加入监测来满足要求。如果 Promise 的 state 大于 0，我们就不再进行通知了：

```js
Deferred.prototype.notify = function(progress) {
  var pending = this.promise.$$state.pending;
  if (pending && pending.length && !this.promise.$$state.status) {
    $rootScope.$evalAsync(function() {
      _.forEach(pending, function(handlers) {
        var progressBack = handlers[3];
        if (_.isFunction(progressBack)) {
          progressBack(progress);
        }
      });
    });
  }
};
```

在 Promise 链中使用 notify 有一些更有趣的特性。首先，就是通知在 Promise 链中是可以冒泡的。只有有一个 Deferred 进行了 notify，在它之后的节点都会收到相同的通知：

```js
it('can notify progress through chain', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();

  d.promise
    .then(_.noop)
    .catch(_.noop)
    .then(null, null, progressSpy);

  d.notify('working...');
  
  $rootScope.$apply();

  expect(progressSpy).toHaveBeenCalledWith('working...');
});
```

要实现这一特性，我们需要利用保存在 pending 数组的 Deferred 对象，它是 pending 数组的第一个元素，也是下一个节点继续执行的钥匙。我们可以在这个 Deferred 上直接调用 notify：

```js
Deferred.prototype.notify = function(progress) {
  var pending = this.promise.$$state.pending;
  if (pending && pending.length &&
    !this.promise.$$state.status) {
    $rootScope.$evalAsync(function() {
      _.forEach(pending, function(handlers) {
        var deferred = handlers[0];
        var progressBack = handlers[3];
        if (_.isFunction(progressBack)) {
          progressBack(progress);
        }
        deferred.notify(progress);
      });
    });
  }
};
```

更有趣的是，我们可能会在 Promise 链中的多个节点中绑定 notify 回调，上一个节点中绑定的 notify 回调的结果值，会成为下一个节点的参数。因此，我们可以随时在节点中改变要传递的信息：

```js
it('transforms progress through handlers', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  d.promise
    .then(_.noop)
    .then(null, null, function(progress) {
      return '***' + progress + '***';
    })
    .catch(_.noop)
    .then(null, null, progressSpy);
  d.notify('working...');
  $rootScope.$apply();
  expect(progressSpy).toHaveBeenCalledWith('***working...***');
});
```

我们可以简单地通过判断 notify 回调是否是函数来判断，使用该回调的返回值还是原始信息作为通知内容：

```js
Deferred.prototype.notify = function(progress) {
  var pending = this.promise.$$state.pending;
  if (pending && pending.length &&
    !this.promise.$$state.status) {
    $rootScope.$evalAsync(function() {
      _.forEach(pending, function(handlers) {
        var deferred = handlers[0];
        var progressBack = handlers[3];
        deferred.notify(_.isFunction(progressBack) ?
          progressBack(progress) :
          progress
        );
      });
    });
  }
};
```

当 Promise 的某个处理器的 notify 回调抛出异常时，是不会影响该 Promise 的其他处理器的 notify 运作的：

```js
it('recovers from progressback exceptions', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  var fulflledSpy = jasmine.createSpy();
  d.promise.then(null, null, function(progress) {
    throw 'fail';
  });
  d.promise.then(fulflledSpy, null, progressSpy);
  d.notify('working...');
  d.resolve('ok');
  $rootScope.$apply();
  expect(progressSpy).toHaveBeenCalledWith('working...');
  expect(fulflledSpy).toHaveBeenCalledWith('ok');
});
```

因此我们会使用 try...catch 包裹对 notify 回调的调用代码。当异常发生时，我们只是把异常打印出来：

```js
Deferred.prototype.notify = function(progress) {
  var pending = this.promise.$$state.pending;
  if (pending && pending.length &&
    !this.promise.$$state.status) {
    $rootScope.$evalAsync(function() {
      _.forEach(pending, function(handlers) {
        var deferred = handlers[0];
        var progressBack = handlers[3];
        try {
          deferred.notify(_.isFunction(progressBack) ?
            progressBack(progress) :
            progress
          );
        } catch (e) {
          console.log(e);
        }
      });
    });
  }
};
```

> 从这里的实现我们还可以知道，如果发生了异常，那么我们不会让下面的 Promise 链节点继续执行 notify 回调，也就是说 notify 将不会进行“冒泡”。但是在发生了异常的这个节点中，对它绑定的其他处理器也会继续生效，上面的测试用例已经覆盖到这种情况了。

这个通知机制同样适用于异步的 Promise 处理器。内层的 Promise 调用了 notify 方法，而外层 Promise 也有 notify 回调的情况下，外层 Promise 的 notify 回调就会因为冒泡机制而被调用：

```js
it('can notify progress through promise returned from handler', function() {
  var d = $q.defer();

  var progressSpy = jasmine.createSpy();
  d.promise.then(null, null, progressSpy);

  var d2 = $q.defer();
  // Resolve original with nested promise
  d.resolve(d2.promise);
  // Notify on the nested promise
  d2.notify('working...');

  $rootScope.$apply();

  expect(progressSpy).toHaveBeenCalledWith('working...');
});
```

跟之前我们实现 resolve 和 reject 方法的“冒泡”类似，我们只需要对内部 Promise 绑定外部 notify 回调作为自己的第三个回调，就可以完成需求：

```js
Deferred.prototype.resolve = function(value) {
  if (this.promise.$$state.status) {
    return;
  }
  if (value && _.isFunction(value.then)) {
    value.then(
      _.bind(this.resolve, this),
      _.bind(this.reject, this),
      _.bind(this.notify, this)
    );
  } else {
    this.promise.$$state.value = value;
    this.promise.$$state.status = 1;
    scheduleProcessQueue(this.promise.$$state);
  }
};
```

我们还可以在 finally 回调中使用 notify 进行通知，我们可以在 finally 函数调用时传入第二个参数：

```js
it('allows attaching progressback in finally', function() {
  var d = $q.defer();
  var progressSpy = jasmine.createSpy();
  d.promise.finally(null, progressSpy);

  d.notify('working...');
  $rootScope.$apply();

  expect(progressSpy).toHaveBeenCalledWith('working...');
});
```

在 finally 内部，由于 then 已经支持了传递 notify 回调，我们就可以直接把 notify 回调作为参数传递给 then 方法的调用：

```js
Promise.prototype.fnally = function(callback, progressBack) {
  return this.then(function(value) {
    return handleFinallyCallback(callback, value, true);
  }, function(rejection) {
    return handleFinallyCallback(callback, rejection, false);
  }, progressBack);
};
```
### 异常处理（Exception Handling）

除了显式地拒绝一个 Promise，promise 回调函数也可能会出现异常。当回调函数中抛出异常时，我们应该让后面绑定了失败回调的节点对异常进行处理：

```js
it('rejects chained promise when handler throws', function() {
  var d = $q.defer();
  
  var rejectedSpy = jasmine.createSpy();
  d.promise.then(function() {
    throw 'fail';
  }).catch(rejectedSpy);
  d.resolve(42);

  $rootScope.$apply();

  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

我们会在 processQueue 里面处理异常的抛出，具体就是我们调用回调的地方。如果异常发生了，将会被下一个 Deferred 的失败回调捕获到：

```js
function processQueue(state) {
  var pending = state.pending;
  state.pending = undefned;
  _.forEach(pending, function(handlers) {
    var deferred = handlers[0];
    var fn = handlers[state.status];
    try {
      if (_.isFunction(fn)) {
        deferred.resolve(fn(state.value));
      } else if (state.status === 1) {
        deferred.resolve(state.value);
      } else {
        deferred.reject(state.value);
      }
    } catch (e) {
      deferred.reject(e);
    }
  });
}
```

要注意这个异常只会被下一个节点捕获，当前抛出异常的 Promise 的失败回调并不会被调用。要知道当我们执行一个 Promise 回调时，对应的 Deferred 肯定已经被 resolve （或 reject）了，我们并不能回退甚至改变这个结果状态。也就是说，下面这个测试用例应该也会被通过：

```js
it('does not reject current promise when handler throws', function() {
  var d = $q.defer();

  var rejectedSpy = jasmine.createSpy();
  d.promise.then(function() {
    throw 'fail';
  });
  d.promise.catch(rejectedSpy);
  d.resolve(42);

  $rootScope.$apply();
  
  expect(rejectedSpy).not.toHaveBeenCalled();
});
```

跟之前那个测试用例相比，这个用例直接在原始的 promise 绑定失败回调，而不是在成功回调后面绑定回调形成链式调用。这个区别是非常关键的。


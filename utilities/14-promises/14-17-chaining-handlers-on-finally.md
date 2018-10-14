### 链式操作中的 finally 处理块（Chaining Handlers on finally）

之前我们介绍了 finally 处理块的实现，同时也留下了要支持链式操作的“坑”。现在我们就来实现支持链式操作的 finally 处理块。

finally 处理块的一个重要特性是——它的返回值将会被忽略。finally 处理块被设计成只做清理工作，而不会参与处理链式操作中的结果值。这意味着 finally 会直接把上一个流程的计算值通过 resolve 往下传递，其中对计算值的操作就将会被跳过：

```js
it('allows chaining handlers on finally, with original value', function() {
  var d = $q.defer();

  var fulflledSpy = jasmine.createSpy();
  d.promise.then(function(result) {
    return result + 1;
  }).finally(function(result) {
    return result * 2;
  }).then(fulflledSpy);
  d.resolve(20);

  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith(21);
});
```

当处理结果为 rejection 时，finally 处理块的行为是类似的，finally 处理块会把 reject 的状态和值直接往下传递：

```js
it('allows chaining handlers on finally, with original rejection', function() {
  var d = $q.defer();

  var rejectedSpy = jasmine.createSpy();
  d.promise.then(function(result) {
    throw 'fail';
  }).finally(function() {}).catch(rejectedSpy);
  d.resolve(20);

  $rootScope.$apply();

  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

在 finally 的实现代码中，我们会在 then 中直接返回上一个节点的原始参数值，而 finally 处理块的回调函数的返回值将会被忽略：

```js
Promisei.prototype.finally = function(callback) {
  return this.then(function(value) {
    callback();
    return value;
  }, function() {
    callback();
  });
};
```

由于 finally 块中也可能会进行异步的操作，所以我们也应该要支持 finally 回调返回 Promise 的情况。如果 finally 返回一个 Promise，我们需要等待这个 Promise 被 resolve 后才继续进行下一个节点的处理，当然，finally 中对要传递的结果值的处理依然会被忽略：

```js
it('resolves to orig value when nested promise resolves', function() {
  var d = $q.defer();

  var fulflledSpy = jasmine.createSpy();
  var resolveNested;

  d.promise.then(function(result) {
    return result + 1;
  }).fnally(function(result) {
    var d2 = $q.defer();
    resolveNested = function() {
      d2.resolve('abc');
    };
    return d2.promise;
  }).then(fulflledSpy);
  d.resolve(20);

  $rootScope.$apply();
  expect(fulflledSpy).not.toHaveBeenCalled();

  resolveNested();

  $rootScope.$apply();
  expect(fulflledSpy).toHaveBeenCalledWith(21);
});
```

这里我们测试了当 finally 块中有异步操作时，该 Promise 节点不会马上被 resolve，只有在异步工作结束后才会继续，异步工作中产生的值将会被忽略。

同样地，对于 rejection 的情况我们也用同样的处理手段：

```js
it('rejects to original value when nested promise resolves', function() {
  var d = $q.defer();

  var rejectedSpy = jasmine.createSpy();
  var resolveNested;

  d.promise.then(function(result) {
    throw 'fail';
  }).fnally(function(result) {
    var d2 = $q.defer();
    resolveNested = function() {
      d2.resolve('abc');
    };
    return d2.promise;
  }).catch(rejectedSpy);
  d.resolve(20);

  $rootScope.$apply();
  expect(rejectedSpy).not.toHaveBeenCalled();

  resolveNested();
  $rootScope.$apply();
  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

只有一种情况下，我们不会忽略 finally 回调的结果值，那就是 finally 结果为 rejection 的情况。在这种情况下，我们会忽略上一个节点传递过来的结果值。这也是因为我们不希望在做清理过程中发生的错误被忽略：

```js
it('rejects when nested promise rejects in fnally', function() {
  var d = $q.defer();

  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();
  var rejectNested;

  d.promise.then(function(result) {
    return result + 1;
  }).fnally(function(result) {
    var d2 = $q.defer();
    rejectNested = function() {
      d2.reject('fail');
    };
    return d2.promise;
  }).then(fulflledSpy, rejectedSpy);
  d.resolve(20);

  $rootScope.$apply();
  expect(fulflledSpy).not.toHaveBeenCalled();

  rejectNested();
  $rootScope.$apply();
  expect(fulflledSpy).not.toHaveBeenCalled();
  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

一旦我们获取 finally 绑定的回调函数，我们需要判断回调函数的返回值是不是一个类 Promise 的对象。如果是的话，我们会为它绑定一个处理器，这个处理器有一个成功回调，这个成功回调总是会返回原始的结果值；这个处理器没有绑定失败回调，这意味着 finally 的 rejection 都会向上传递：

```js
Promise.prototype.finally = function(callback) {
  return this.then(function(value) {
    var callbackValue = callback();
    if (callbackValue && callbackValue.then) {
      return callbackValue.then(function() {
        return value;
      });
    } else {
      return value;
    }
  }, function(rejection) {
    callback();
    var d = new Deferred();
    d.reject(rejection);
    return d.promise;
  });
};
```

对于原始值为 rejection 的情况，我们也要做类似的处理：如果 finally 回调返回一个 Promise，我们会等待 Promise 有结果后再将 rejection 的状态值往下传递。无论是原始值是一个 rejection，还是 finally 中的 Promise 返回 rejection，我们会向下传递一个被拒绝后的 Promise：

```js
Promise.prototype.fnally = function(callback) {
  return this.then(function(value) {
    var callbackValue = callback();
    if (callbackValue && callbackValue.then) {
      return callbackValue.then(function() {
        return value;
      });
    } else {
      return value;
    }
  }, function(rejection) {
    var callbackValue = callback();
    if (callbackValue && callbackValue.then) {
      return callbackValue.then(function() {
        var d = new Deferred();
        d.reject(rejection);
        return d.promise;
      });
    } else {
      var d = new Deferred();
      d.reject(rejection);
      return d.promise;
    }
  });
};
```

现在我们已经实现了完整版的 finally 了，但目前的实现还是有点冗长。下面我们通过封装几个帮助函数来简化流程：

首先，我们可以使用一个通用的帮助函数，这个函数可以接收一个原始结果值和布尔值标识，并根据布尔值标识返回一个带上原始结果值的已经 resolve 或 reject 的 Promise 对象。

```js
function makePromise(value, resolved) {
  var d = new Deferred();
  if (resolved) {
    d.resolve(value);
  } else {
    d.reject(value);
  }
  return d.promise;
}
```

现在我们可以在 finally 的代码实现中使用这个帮助函数：

```js
Promise.prototype.fnally = function(callback) {
  return this.then(function(value) {
    var callbackValue = callback();
    if (callbackValue && callbackValue.then) {
      return callbackValue.then(function() {
        return makePromise(value, true);
      });
    } else {
      return value;
    }
  }, function(rejection) {
    var callbackValue = callback();
    if (callbackValue && callbackValue.then) {
      return callbackValue.then(function() {
        return makePromise(rejection, false);
      });
    } else {
      return makePromise(rejection, false);
    }
  });
};
```

我们还可以对 finally 中 then 绑定的两个回调函数进行封装，毕竟这两个回调函数的逻辑都基本一致：

```js
Promise.prototype.fnally = function(callback) {
  return this.then(function(value) {
    return handleFinallyCallback(callback, value, true);
  }, function(rejection) {
    return handleFinallyCallback(callback, rejection, false);
  });
};
```

这个帮助函数会调用 finally 回调，然后返回一个与原始结果值相同的 Promise，除非 finally 回调本身返回结果值为 rejection:

```js
function handleFinallyCallback(callback, value, resolved) {
  var callbackValue = callback();
  if (callbackValue && callbackValue.then) {
    return callbackValue.then(function() {
      return makePromise(value, resolved);
    });
  } else {
    return makePromise(value, resolved);
  }
}
```

综上，当 finally 返回值是一个 Promise 时，我们会等待 Promise 完成后才往下继续处理流程。除非 finally 内部的 Promise 被 reject 了，否则我们都会直接把原始结果值往下传递。
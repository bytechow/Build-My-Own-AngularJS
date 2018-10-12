### Promise 链（Promise Chaining）

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

我们还得把 Defferred  对应的计算结果放到 onFulfilled 回调被调用的地方才行，这样才能实现传递。我们会把这个计算结果作为 $$state.pending 第一个参数：

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

在之后我们要在 processQueue 中执行回调时，就可以把计算结果拿出来，并传递给对应的回调函数：

```js
unction processQueue(state) {
  var pending = state.pending;
  state.pending = undefned;
  _.forEach(pending, function(handlers) {
    var deferred = handlers[0];
    var fn = handlers[state.status];
    if (_.isFunction(fn)) {
      deferred.resolve(fn(state.value));
    }
  });
}
```

这就是我们可以使用 then 形成 Promise 链的关键。每一个 Promise 处理节点都会创建一个新的 Deferred，同时也意味着创建了一个 Promise。这个新的 Deferred 与原始的 Deferred 是完全独立的，但只要原始的 Deferred 被决议，新的 Deferred 也会跟随着变成被决议的状态。

当然，由于每一个 then 都会创建一个新的 Deferred 并返回一个 Promise，所以在 Promise 处理链条的最后一个节点也会返回一个 Promise，但这个 Promise 自然会被忽略掉。

Promise 链的另一个特性是，计算结果的状态和值都会被向下传递。比如，我们拒绝了一个 Deferred，然后在 Promise 链的下一个流程只注册成功后回调（不注册失败回调），到再下一个流程我们才绑定一个失败回调。由于在 Promise 链中，拒绝（rejection）这个状态和值都会一直向下传递，所以我们可以实现`one.then(two).catch(three)`这样的结构，而`three`这个流程可以捕捉来自`one`或者`two`的错误：

```js
it('catches rejection on chained handler', function() {
  var d = $q.defer();

  var rejectedSpy = jasmine.createSpy();
  d.promise.then(_.noop).catch(rejectedSpy);

  d.reject('fail');
  $rootScope.$apply();

  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

当然，这个特性不仅仅适用于 Deferred 被拒绝（rejection）的情况，对于 Deferred 被解决（resolution）的情况也同样适用。当某个 Promise 链只注册了失败回调，但是 Deferred 的计算结果是“已解决”的，这个结果值就会被往下传递，直到 Promise 处理链条中有一个 Promise 注册了成功回调，这个结果值会作为这个成功回调的参数被调用：

```js
it('fulflls on chained handler', function() {
  var d = $q.defer();
  var fulflledSpy = jasmine.createSpy();
  d.promise.catch(_.noop).then(fulflledSpy);
  d.resolve(42);
  $rootScope.$apply();
  expect(fulflledSpy).toHaveBeenCalledWith(42);
});
```

由于在上面的测试中使用到了`_.noop`，我们需要在`test/q_spec.js`引入 LoDash：

```js
'use strict';

var _ = require('lodash');
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
```

上面所说的功能，我们可以通过对 processQueue 函数进行小小的修改就可以满足。在每个 Promise 回调函数被执行时，如果当前 Promise 没有注册对应状态的回调函数，我们就直接把当前 Promise 的状态和值向下传递：

```js
function processQueue(state) {
  var pending = state.pending;
  state.pending = undefned;
  _.forEach(pending, function(handlers) {
    var deferred = handlers[0];
    var fn = handlers[state.status];
    if (_.isFunction(fn)) {
      deferred.resolve(fn(state.value));
    } else if (state.status === 1) {
      deferred.resolve(state.value);
    } else {
      deferred.reject(state.value);
    }
  });
}
```

链式操作还有一点需要注意，当某个 Deferred 被拒绝，下一个 catch 回调就会捕捉到这个拒绝状态值，而这个 catch 回调的返回值将会被看作 resolution，而不是 rejection。其实，通过上面的代码改进，我们已经实现了这个特性，只是还不是很明显而已。这里的巧妙在于，只要 Promise 有注册回调函数，无论 Deferred 是被解决还是被拒绝，都会使用`deferred.resolve`进行处理。

其实，这也与我们传统的`try...catch...`语法有异曲同工之处，catch 代码块会处理错误，而不再将错误向外抛出，之后的程序还是会被正常执行。

```js
it('treats catch return value as resolution', function() {
  var d = $q.defer();

  var fulflledSpy = jasmine.createSpy();
  d.promise
    .catch(function() {
      return 42;
    })
    .then(fulflledSpy);

  d.reject('fail');
  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith(42);
});
```




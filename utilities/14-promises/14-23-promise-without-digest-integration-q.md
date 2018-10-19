## 不需要集成 $digest 的 Promise：$$q（Promise Without $digest Integration: $$q）

在本章的最后，我们会介绍一种有趣却很少知道的 Angular 服务——$$q 服务。

和 $q 服务一样，这是一种 Promise 的实现。不同的是，我们在 $$q 服务中不再集成 $rootScope digest，而会使用浏览器的 timeout 机制完成异步操作。换句话说，$$q 更贴近于独立于 AngularJS 的 Promise 实现。

> 从它拥有两个 $ 符号，可以看出 $$q 服务是 Angular 中的内部服务，如果应用开发者使用的话，将会报出一些警告：$$q 可能会在之后的版本中被移除或更改而不会进行提示，在升级 Angular 版本后，继续使用该服务可能会令你的应用变得不可用。

$$q 在 Angular 内部可以被应用于 $timeout 和 $interval 服务，只要在调用它们的时候加入\`skipApply\` 标识参数即可，也会被 ngAnimate 用于它的异步操作。

在我们开始测试 $$q 之前，我们需要将它放到 $q 的测试文件中，所以我们会从 injector 中获取它。这会对当前大部分测试单元产生影响：

```js
var $q, $$q, $rootScope;

beforeEach(function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  $q = injector.get('$q');
  $$q = injector.get('$$q');
  $rootScope = injector.get('$rootScope');
});
```

要修复这些测试用例，我们会在 src/q.js 中增加一个针对 $$q 的 provider。并把它和 $q 服务对应的 provider 向外抛出：

```js
function $$QProvider() {
  this.$get = function() {

  };
}
module.exports = {
  $QProvider: $QProvider,
  $$QProvider: $$QProvider
};
```

然后，我们可以将 $$q 服务发布到 ng 模块上：

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  
  var ngModule = window.angular.module('ng', []);
  ngModule.provider('$filter', require('./filter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
  ngModule.provider('$q', require('./q').$QProvider);
  ngModule.provider('$$q', require('./q').$$QProvider);
}
```

准备就绪，我们就可以加入 $$q 的第一个测试用例了。在这个用例中，我们会测试它是否能像 $q 一样正常产出 Deferred 和 Promise，但并不会在 digest 循环结束后就被 resolve：

```js
describe('$$q', function() {

  it('uses deferreds that do not resolve at digest', function() {
    var d = $$q.defer();
    var fulfilledSpy = jasmine.createSpy();
    d.promise.then(fulfilledSpy);
    d.resolve('ok');
    
    $rootScope.$apply();
    
    expect(fulfilledSpy).not.toHaveBeenCalled();
  });
});
```

相反，Promise 会在一段时间结束后被 resolve。我们可以使用 Jasmine 的 fake clock 功能来辅助测试。它可以模拟一段时间过去之后发生的事情：

```js
describe('$$q', function() {

  beforeEach(function() {
    jasmine.clock().install();
  });
  afterEach(function() {
    jasmine.clock().uninstall();
  });
  
  it('uses deferreds that do not resolve at digest', function() {
    var d = $$q.defer();
    var fulfilledSpy = jasmine.createSpy();
    d.promise.then(fulfilledSpy);
    d.resolve('ok');
    
    $rootScope.$apply();
    
    expect(fulfilledSpy).not.toHaveBeenCalled();
  });
  
  it('uses deferreds that resolve later', function() {
    var d = $$q.defer();
    var fulfilledSpy = jasmine.createSpy();
    d.promise.then(fulfilledSpy);
    d.resolve('ok');
    
    jasmine.clock().tick(1);
    
    expect(fulfilledSpy).toHaveBeenCalledWith('ok');
  });
});
```

这里，我们模拟时间过去了 1 毫秒，发现 Promise 果然被 resolve 了。

当然 $$q 还有一个关乎性能的重要特性，就是当它内部的 Promise 被 resolve 后，它不会触发 digest 循环：

```js
it('does not invoke digest', function() {
  var d = $$q.defer();
  d.promise.then(_.noop);
  d.resolve('ok');
  
  var watchSpy = jasmine.createSpy();
  $rootScope.$watch(watchSpy);
  
  jasmine.clock().tick(1);
  
  expect(watchSpy).not.toHaveBeenCalled();
});
```

那我们怎么来实现 $$q 服务呢？需要再拷贝一份 $q 的实现逻辑吗？当然不需要。实际上，$q 和 $$q 的区别就在于 Deferred 是用哪种方式产生异步的：$q 使用 $evalAsync，而 $$q 使用 setTimeout。

所以我们要做的是，把现在 $QProvide.$get 中的代码用一个帮助函数——qFactory 来进行包裹。我们可以把异步策略作为参数调用 qFactory，下面是具体的实现代码：

```js
function $QProvider() {
  this.$get = ['$rootScope', function($rootScope) {
    return qFactory(function(callback) {
      $rootScope.$evalAsync(callback);
    });
  }];
}

function $$QProvider() {
  this.$get = function() {
    return qFactory(function(callback) {
      setTimeout(callback, 0);
    });
  };
}
```

之前 $q 的初始化代码现在都放到 qFactory 里面了，只不过之前使用 $evalAsync 进行异步调用的地方，现在都用 callLater 函数取代了.。下面我们给出最终的 q.js 源代码：

```js
'use strict';
var _ = require('lodash');

function qFactory(callLater) {
  function processQueue(state) {
    var pending = state.pending;
    state.pending = undefined;
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

  function scheduleProcessQueue(state) {
    callLater(function() {
      processQueue(state);
    });
  }

  function makePromise(value, resolved) {
    var d = new Deferred();
    if (resolved) {
      d.resolve(value);
    } else {
      d.reject(value);
    }
    return d.promise;
  }

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

  function Promise() {
    this.$$state = {};
  }
  Promise.prototype.then = function(onFulfilled, onRejected, onProgress) {
    var result = new Deferred();
    this.$$state.pending = this.$$state.pending || [];
    this.$$state.pending.push([result, onFulfilled, onRejected, onProgress]);
    if (this.$$state.status > 0) {
      scheduleProcessQueue(this.$$state);
    }
    return result.promise;
  };
  Promise.prototype.catch = function(onRejected) {
    return this.then(null, onRejected);
  };
  Promise.prototype.finally = function(callback, progressBack) {
    return this.then(function(value) {
      return handleFinallyCallback(callback, value, true);
    }, function(rejection) {
      return handleFinallyCallback(callback, rejection, false);
    }, progressBack);
  };

  function Deferred() {
    this.promise = new Promise();
  }
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
  Deferred.prototype.reject = function(reason) {
    if (this.promise.$$state.status) {
      return;
    }
    this.promise.$$state.value = reason;
    this.promise.$$state.status = 2;
    scheduleProcessQueue(this.promise.$$state);
  };
  Deferred.prototype.notify = function(progress) {
    var pending = this.promise.$$state.pending;
    if (pending && pending.length &&
      !this.promise.$$state.status) {
      callLater(function() {
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

  function defer() {
    return new Deferred();
  }

  function reject(rejection) {
    var d = defer();
    d.reject(rejection);
    return d.promise;
  }

  function when(value, callback, errback, progressback) {
    var d = defer();
    d.resolve(value);
    return d.promise.then(callback, errback, progressback);
  }

  function all(promises) {
    var results = _.isArray(promises) ? [] : {};
    var counter = 0;
    var d = defer();
    _.forEach(promises, function(promise, index) {
      counter++;
      when(promise).then(function(value) {
        results[index] = value;
        counter--;
        if (!counter) {
          d.resolve(results);
        }
      }, function(rejection) {
        d.reject(rejection);
      });
    });
    if (!counter) {
      d.resolve(results);
    }
    return d.promise;
  }
  
  var $Q = function Q(resolver) {
    if (!_.isFunction(resolver)) {
      throw 'Expected function, got ' + resolver;
    }
    var d = defer();
    resolver(
      _.bind(d.resolve, d),
      _.bind(d.reject, d)
    );
    return d.promise;
  };

  return _.extend($Q, {
    defer: defer,
    reject: reject,
    when: when,
    resolve: when,
    all: all
  });
}

function $QProvider() {
  this.$get = ['$rootScope', function($rootScope) {
    return qFactory(function(callback) {
      $rootScope.$evalAsync(callback);
    });
  }];
}

function $$QProvider() {
  this.$get = function() {
    return qFactory(function(callback) {
      setTimeout(callback, 0);
    });
  };
}

module.exports = {
  $QProvider: $QProvider,
  $$QProvider: $$QProvider
};
```

只是一些简单的函数组合起来，就带给我如此有用的特性！并且我们一下子就多了两个服务呢。


### 遵循 ES2015 规范的 Promise（ES2015-Style Promises）

正如我们在本章开头提到了，ES2015 内置了 Promise 的实现。按照我们目前的实现，已经能够将 $q 服务和标准的 Promise 实现进行友好的结合了，因为我们目前在 $q 采用可链式操作的 Promise 对象，ES2015也在使用。

如果你想要更进一步，创建更加接近标准的 Promise 对象，$q 服务也可以很好地进行支持。其实 ES2015 支持的 Promise 实现跟我们在 Angular 中的相比，最大的差别就在于 Angular 中明确提出了 Deferred 的概念（并在代码中有所体现）。而 ES2015 的实现方式是：创建一个 Promise 实例，并在创建时提供一个函数作为参数。这个函数接收两个回调函数即 resolve 和 reject，当完成操作后，我们可以按照结果选择 resolve 或者 reject。

如果要使用 ES6 的 Promise，我们需要将下面代码：

```js
var deferred = Q.defer();
doAsyncStuff(function(err) {
  if (err) {
    deferred.reject(err);
  } else {
    deferred.resolve();
  }
});
return deferred.promise;
```

替换为下面的形势：

```js
return new Promise(function(resolve, reject) {
  doAsyncStuff(function(err) {
    if (err) {
      reject(err)
    } else {
      resolve();
    }
  });
});
```

所以我们可以看出，实际上 Deferred 对象将变成一个嵌套的函数，而只保留一个对外的 API 接口——Promise。

下面我们来看一下 $q 服务时如何提供类似的 API 的，首先，$q 在实际上会变成一个函数：

```js
describe('ES2015 style', function() {
  it('is a function', function() {
    expect($q instanceof Function).toBe(true);
  });
});
```

我们将在 provider 中创建一个内部函数 $Q，然后把所有之前我们队外暴露的 Deferrer 方法接口都绑定到 $Q 上去：

```js
ar $Q = function Q() {
  
};
return _.extend($Q, {
  defer: defer,
  reject: reject,
  when: when,
  resolve: when,
  all: all
});
```

像 ES2015 Promise 构造函数一样，这个新函数也需要有一个函数作为参数，我们就叫它的 resolver 函数吧，而且这个参数是必须的：

```js
it('expects a function as an argument', function() {
  expect($q).toThrow();
  $q(_.noop); // Just checking that this doesn't throw
});
```

```js
var $Q = function Q(resolver) {
  if (!_.isFunction(resolver)) {
    throw 'Expected function, got ' + resolver;
  }
};
```

这个函数的返回值就是一个 Promise：

```js
it('returns a promise', function() {
  expect($q(_.noop)).toBeDefned();
  expect($q(_.noop).then).toBeDefned();
});
```

在 $Q 函数内部，我们可以使用 defer 函数来实现返回一个 Promise 实例：

```js
var $Q = function Q(resolver) {
  if (!_.isFunction(resolver)) {
    throw 'Expected function, got ' + resolver;
  }
  var d = defer();

  return d.promise;
};
```

就像我们上面介绍的 ES2015 Promise 例子，resolver 函数内可以调用 resolve 回调，当 resolve 回调被调用，Promise 也就随之被 resolve 了：

```js
it('calls function with a resolve function', function() {
  var fulflledSpy = jasmine.createSpy();
  
  $q(function(resolve) {
    resolve('ok');
  }).then(fulflledSpy);
  
  $rootScope.$apply();
  
  expect(fulflledSpy).toHaveBeenCalledWith('ok');
});
```

我们可以直接把内部 Deferred 的 resolve 方法作为 resolver 函数的第一个参数，并且保证resolve 回调的 this 上下文指向这个 Deferred，我们会使用 \_.bind：

```js
var $Q = function Q(resolver) {
  if (!_.isFunction(resolver)) {
    throw 'Expected function, got ' + resolver;
  }
  var d = defer();
  resolver(_.bind(d.resolve, d));
  return d.promise;
};
```

对于 rejection 的情况，我们会把 reject 回调作为第二个参数。如果 reject 回调被调用，Promise 也会随之变成 reject 状态：

```js
it('calls function with a reject function', function() {
  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();
  
  $q(function(resolve, reject) {
    reject('fail');
  }).then(fulflledSpy, rejectedSpy);

  $rootScope.$apply();

  expect(fulflledSpy).not.toHaveBeenCalled();
  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

对于 reject 回调的实现，其实与 resolve 回调相差无几：

```js
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
```

就这样，我们就可以在 $q 服务中实现类似 ES2015 的调用方式！可以看到，实际上我们还是借用了本章实现的 Deferred，只不过实现细节都封装好了。如果你习惯使用 ES2015 的调用方式，这个功能会帮到你！

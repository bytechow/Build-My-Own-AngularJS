### 处理 Promise 集合——$q.all（Working with Promise Collections）

当有多个异步任务要执行时，将它们打包成一个 Promise 集合会变得很有用。Promise 说白了就是一个 JavaScript 对象而已，所以实际上你可以用任何可以生成、操作和转换集合的函数或库对它们进行处理。

如果有专门针对 Promise的集合方法，这些方法可以组合并处理异步任务，在某些情况下会更有用。社区已经有相关的解决方案了，比如[async-q](https://www.npmjs.com/package/async-q)。然而，Angular 已经提供了一个专门用于组合并处理 Promise 集合的方法了，它就是——$q.all。

$q.all 方法接收一个 Promise 集合作为它的参数，返回值就是集合中各个 Promise 的计算结果值。结果数组的元素与集合中的 Promise 一一对应，这对于同时进行的异步任务来说尤其有用:

```js
describe('all', function() {

  it('can resolve an array of promises to array of results', function() {
    var promise = $q.all([$q.when(1), $q.when(2), $q.when(3)]);
    var fulflledSpy = jasmine.createSpy();
    promise.then(fulflledSpy);

    $rootScope.$apply();
function all(promises) {
  var results = _.isArray(promises) ? [] : {};
  var counter = 0;
  var d = defer();
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
      if (!counter) {
        d.resolve(results);
      }
    });
  });
  return d.promise;
}    expect(fulflledSpy).toHaveBeenCalledWith([1, 2, 3]);
  });
});
```

显然，这也是一个新的 API：

```js
function all(promises) {

}

return {
  defer: defer,
  reject: reject,
  when: when,
  resolve: when,
  all: all
};
```

这个函数会创建一个存放计算结果的数据，并为每一个 Promise 增加一个回调函数，结果数组的元素位置与 Promise 的位置一一对应：

```js
function all(promises) {
  var results = [];
  _.forEach(promises, function(promise, index) {
    promise.then(function(value) {
      results[index] = value;
    });
  });
}
```

在这个函数里面还有一个内部变量充当计数器，每一次循环都将会加1，而如果有 Promise 回调被调用就会减 1:

```js
function all(promises) {
  var results = [];
  var counter = 0;
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
    });
  });
}
```

这意味着如果计数器变成0，则所有的 Promise 都已经被 resolve 了。如果我们在里面构造一个 Deferred，并让它在计数器变成 0 后 resolve，就完成 $q.all 的逻辑了：

```js
function all(promises) {
  var results = [];
  var counter = 0;
  var d = defer();
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
      if (!counter) {
        d.resolve(results);
      }
    });
  });
  return d.promise;
}
```

$q.all 不仅可以传入数组，还支持传入对象作为参数。如果传入对象，那么结果也将会是一个对象，对象的属性值就是对应 Promise 的结果值：

```js
it('can resolve an object of promises to an object of results', function() {
  var promise = $q.all({
    a: $q.when(1),
    b: $q.when(2)
  });
  var fulflledSpy = jasmine.createSpy();
  promise.then(fulflledSpy);

  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith({
    a: 1,
    b: 2
  });
});
```

因此，在 all 方法的一开始，我们就要判断参数是一个数组还是对象，并以此决定结果的输出。这里我们只需要进行参数判断即可，\_.forEach 可以兼容进行数组或对象类型的遍历：

```js
function all(promises) {
  var results = _.isArray(promises) ? [] : {};
  var counter = 0;
  var d = defer();
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
      if (!counter) {
        d.resolve(results);
      }
    });
  });
  return d.promise;
}
```

目前的 $q.all 还有一个问题，如果传入的 promises 参数为空数据，那么 Promise 将永远不会被 resolve。我们期望在这种情况下 $q.all 的返回值也变成空数组，也意味着 Promise 会被 resolve 掉。

```js
it('resolves an empty array of promises immediately', function() {
  var promise = $q.all([]);
  var fulflledSpy = jasmine.createSpy();
  promise.then(fulflledSpy);

  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith([]);
});
```

同样，如果传入参数是一个空对象，我们也如法炮制：

```js
it('resolves an empty object of promises immediately', function() {
  var promise = $q.all({});
  var fulflledSpy = jasmine.createSpy();
  promise.then(fulflledSpy);

  $rootScope.$apply();
  
  expect(fulflledSpy).toHaveBeenCalledWith({});
});
```

我们需要对这两种情况进行特殊处理。当遍历完 Promise 之后，我们需要增加一个额外的检查：如果计数器已经是0了，我们可以立即将结果 resolve 掉。

```js
function all(promises) {
  var results = _.isArray(promises) ? [] : {};
  var counter = 0;
  var d = defer();
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
      if (!counter) {
        d.resolve(results);
      }
    });
  });
  if (!counter) {
    d.resolve(results);
  }
  return d.promise;
}
```

但我们也清楚，不是所有事情都会那么顺利，Promise 也可能会被拒绝。如果 Promise 集合中的一个被拒绝的话，会怎么样？

实际上，如果集合中有一个 Promise 被拒绝的话，整个 Promise 集合都会变成拒绝状态，之前已经被 resolve 的计算结果也会被丢弃：

```js
it('rejects when any of the promises rejects', function() {
  var promise = $q.all([$q.when(1), $q.when(2), $q.reject('fail')]);
  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();
  promise.then(fulflledSpy, rejectedSpy);

  $rootScope.$apply();
  
  expect(fulflledSpy).not.toHaveBeenCalled();
  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

我们可以为集合中的每个 Promise 加入一个失败回调，如果其中一个失败回调被调用，那么整个集合都会被 reject 掉，并带上当前 Promise 的 reject 原因:

```js
function all(promises) {
  var results = _.isArray(promises) ? [] : {};
  var counter = 0;
  var d = defer();
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
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
```

只差最后一点，我们就可以完整实现 $q.all 了。我们并不能限制所有 Promise 集合中元素都一定是 Promise 对象，传入 $q.all 的集合元素也可以是基本数据类型，最终它们会原封不动地传回到结果数组：

```js
it('wraps non-promises in the input collection', function() {
  var promise = $q.all([$q.when(1), 2, 3]);
  var fulflledSpy = jasmine.createSpy();
  promise.then(fulflledSpy);

  $rootScope.$apply();
  
  expect(fulflledSpy).toHaveBeenCalledWith([1, 2, 3]);
});
```

我们可以通过之前实现的 when 方法解决这个问题。when 方法是可以将基本数据类型或者 thenable 对象都转换为 Promise：

```js
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
```

这就是我们需要的 $q.all 了！它不仅实用价值很高，也向我们展示整合 Promise 并不困难。其他集合方法，像 filter 和 reduce 都可以像这样轻松构建。
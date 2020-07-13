### 用一个 listener 函数同时监听多个变化——$watchGroup

#### Watching Several Changes With One Listener: $watchGroup

到目前为止，我们遇到的 watch 函数和 listener 函数都是简单的因果对：当这个发生变化时，就这样做。然而还有另一种常见的情况，那就是同时观察多个状态，当其中一个状态发生改变时就执行某段代码。

实际上，由于 Angular watch 是一个普通的 JavaScript 函数，因此用我们目前已有的 watch 函数就可以实现这个效果：我们可以在 watch 函数中进行多个脏值检查，最后返回触发 listener 函数执行的组合值。

> 译者注：当然这里说的 watcher 需要传入第三个参数值 `true`，开启基于值的比较模式才可以检测复合值内容是否发生变化。

事实上，在 Angular 1.3 以后的版本中我们已经不再需要手动创建这种（一次性监听多个值的）函数，可以直接使用 Angular 内建的 `$watchGroup` 方法。

这个 `$watchGroup` 函数能以数组的形式接受多个 watch 函数和一个 listener 函数。它的核心要点是当数组中任何一个 watch 函数侦测到变化，都会调用 listener 函数。这个 listener 函数的参数是两个数组，分别是新值和旧值，它们会按照传入时的顺序进行排列。

下面，我们新建一个新的 `describe` 代码块来存放对应的测试用例：

_test/scope\_spec.js_

```js
describe('$watchGroup', function() {

  var scope;
  beforeEach(function() {
    scope = new Scope();
  });

  it('takes watches as an array and calls listener with arrays', function() {
    var gotNewValues, gotOldValues;

    scope.aValue = 1;
    scope.anotherValue = 2;

    scope.$watchGroup([
      function(scope) { return scope.aValue; },
      function(scope) { return scope.anotherValue; }
    ], function(newValues, oldValues, scope) {
      gotNewValues = newValues;
      gotOldValues = oldValues;
    });
    scope.$digest();

    expect(gotNewValues).toEqual([1, 2]);
    expect(gotOldValues).toEqual([1, 2]);
  });
});
```

在这个测试中，我们把 listener 函数中传入的 `newValue` 和 `oldValue` 进行捕获，然后检查它们的值是不是由两个 watch 函数的返回值组成的数组。

我们先来试着实现 `$watchGroup`。我们会对每一个 watch 进行单独注册，同时让它们都使用同一个 listener 函数：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  var self = this;
  _.forEach(watchFns, function(watchFn) {
    self.$watch(watchFn, listenerFn);
  });
};
```

但这并不能完全解决问题。我们希望 listener 接收到的是包含全部 watch 返回值的数组，但现在每次 listener 被调用时接收到的只是单个 watch 函数的返回值。

我们需要为每一个 watch 都定义一个内部的 listener 函数，然后在内部 listener 函数呗调用时就把这个 watch 的结果值更新到结果数组的对应位置中去。最后，我们就可以把这个结果数组传递给原始的 listener 函数了。我们需要创建两个数组，一个用于存放新值，一个用于存放旧值：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  // var self = this;
  var newValues = new Array(watchFns.length);
  var oldValues = new Array(watchFns.length);
  _.forEach(watchFns, function(watchFn, i) {
    self.$watch(watchFn, function(newValue, oldValue) {
      newValues[i] = newValue;
      oldValues[i] = oldValue;
      listenerFn(newValues, oldValues, self);
    });
  });
};
```

> `$watchGroup` 用的是中是基于引用的变化侦测方式。

上面第一个版本的 `$watchGroup` 的问题在于我们太急于要调用 listener 函数了：如果侦听器数组有多个 watch 发生了变化，listener 就会被多次调用，而我们希望在这样的情况下 listener 函数只会被调用一次。更糟糕的是，由于程序一旦侦测到变化就会调用一次 listener，因此会在 `oldValues` 和 `newValues` 数组中混合了新值和旧值，导致用户看到前后不一致的结果值。

我们先测试在同时有多个数据发生变化的情况下，listener 函数是否只会被调用一次：

_test/scope\_spec.js_

```js
it('only calls listener once per digest', function() {
  var counter = 0;

  scope.aValue = 1;
  scope.anotherValue = 2;

  scope.$watchGroup([
    function(scope) { return scope.aValue; },
    function(scope) { return scope.anotherValue; }
  ], function(newValues, oldValues, scope) {
    counter++;
  });
  scope.$digest();

  expect(counter).toEqual(1);
});
```

那我们怎么才能把 listener 函数延后到所有 watch 函数都被执行了以后再调用呢？由于 `$watchGroup` 并不会负责启动 digest，适合调用 listener 的位置并没有那么显而易见。但我们可以利用前面章节中实现的 `$evalAsync` 方法。`$evalAsync` 的作用延迟某个函数的执行时间，但保证这个函数依然会在同一个 digest 周期之内执行——这正是我们需要的。

我们会在 `$watchGroup` 中创建一个名为 叫做 `$watchGroupListener` 的内部函数。这个函数负责调用原始的 listener 函数，并传入之前创建的两个数组。在每一个内部 listener 函数中，我们检查一下之前没有设定调用 listener 的定时任务，如果没有，我们就设定一个：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  var self = this;
  var oldValues = new Array(watchFns.length);
  var newValues = new Array(watchFns.length);
  var changeReactionScheduled = false;

  function watchGroupListener() {
    listenerFn(newValues, oldValues, self);
    changeReactionScheduled = false;
  }

  _.forEach(watchFns, function(watchFn, i) {
    self.$watch(watchFn, function(newValue, oldValue) {
      newValues[i] = newValue;
      oldValues[i] = oldValue;
      if (!changeReactionScheduled) {
        changeReactionScheduled = true;
        self.$evalAsync(watchGroupListener);
      }
    });
  });
};
```

以上这就是 `$watchGroup` 的基本内容了，下面我们来关注几种特殊情况。

第一个特殊情况与“listener 第一次调用时传入的新旧值必须为一致”的规范有关。实际上现在的 `$watchGroup` 看似已经是符合我们的要求了，因为它是建立在已经实现这种规范的 `$watch` 函数上。当 listener 函数第一次被调用时，`newValue` 和 `oldValue` 的内容就是一样的了。

虽然这两个数组的内容一样，但它们依然是不同的两个数组对象。这破坏了新旧值必须为同一个值的约束。这也意味着用户比较两个值的时候不能使用基于引用的比较方式，而必须对数组里面的所有内容进行逐一比较，看两个数组的内容是否也是完全相同的。

我们希望能做得更好，让 listener 函数在第一次被调用时传入的新值和旧值是同一个值。

```js
it('uses the same array of old and new values on first run', function() {
  var gotNewValues, gotOldValues;

  scope.aValue = 1;
  scope.anotherValue = 2;

  scope.$watchGroup([
    function(scope) { return scope.aValue; },
    function(scope) { return scope.anotherValue; }
  ], function(newValues, oldValues, scope) {
    gotNewValues = newValues;
    gotOldValues = oldValues;
  });
  scope.$digest();

  expect(gotNewValues).toBe(gotOldValues);
});
```

同时我们也要保证实现这个需求时不会破坏现有的功能。我们可以通过检查 listener 函数在第一次被调用后再次被调用时，传入的新旧值数组是否不同来验证：

_test/scope\_spec.js_

```js
it('uses different arrays for old and new values on subsequent runs', function() {
  var gotNewValues, gotOldValues;

  scope.aValue = 1;
  scope.anotherValue = 2;

  scope.$watchGroup([
    function(scope) { return scope.aValue; },
    function(scope) { return scope.anotherValue; }
  ], function(newValues, oldValues, scope) {
    gotNewValues = newValues;
    gotOldValues = oldValues;
  });
  scope.$digest();

  scope.anotherValue = 3;
  scope.$digest();

  expect(gotNewValues).toEqual([1, 3]);
  expect(gotOldValues).toEqual([1, 2]);
});
```

我们可以通过检查当前 listener 是否是被第一次调用来实现这个需求。如果是第一次调用，那我们就把 `newValues` 同时作为新值和旧值：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  // var self = this;
  // var oldValues = new Array(watchFns.length);
  // var newValues = new Array(watchFns.length);
  // var changeReactionScheduled = false;
  var firstRun = true;

  function watchGroupListener() {
    if (firstRun) {
      firstRun = false;
      listenerFn(newValues, newValues, self);
    } else {
      listenerFn(newValues, oldValues, self);
    }
    // changeReactionScheduled = false;
  }

  // _.forEach(watchFns, function(watchFn, i) {
  //   self.$watch(watchFn, function(newValue, oldValue) {
  //     newValues[i] = newValue;
  //     oldValues[i] = oldValue;
  //     if (!changeReactionScheduled) {
  //       changeReactionScheduled = true;
  //       self.$evalAsync(watchGroupListener);
  //     }
  //   });
  // });
};
```

另外一种特殊情况出现在存放 watch 函数的数组为空时。在这种情况下该怎么做还不是很明显。按照我们目前的实现，在这种情况下程序什么都不会做——没有 watch 函数，就不会调用 listener 函数。实际上，Angular 在遇到这种情况时，也会确保 listener 函数被调用一次，然后把空数组作为结果值（同时作为新值和旧值）：

_test/scope\_spec.js_

```js
it('calls the listener once when the watch array is empty', function() {
  var gotNewValues, gotOldValues;

  scope.$watchGroup([], function(newValues, oldValues, scope) {
    gotNewValues = newValues;
    gotOldValues = oldValues;
  });
  scope.$digest();

  expect(gotNewValues).toEqual([]);
  expect(gotOldValues).toEqual([]);
});
```

我们需要在 `$watchGroup` 方法中检查数组是否为空，如果为空，我们就设定一个调用 listener 函数的延时任务，然后下面的代码就没有必要继续执行了，直接退出就可以了：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  // var self = this;
  // var oldValues = new Array(watchFns.length);
  // var newValues = new Array(watchFns.length);
  // var changeReactionScheduled = false;
  // var firstRun = true;

  if (watchFns.length === 0) {
    self.$evalAsync(function() {
      listenerFn(newValues, newValues, self);
    });
    return;
  }

  // function watchGroupListener() {
  //   if (firstRun) {
  //     firstRun = false;
  //     listenerFn(newValues, newValues, self);
  //   } else {
  //     listenerFn(newValues, oldValues, self);
  //   }
  //   changeReactionScheduled = false;
  // }

  // _.forEach(watchFns, function(watchFn, i) {
  //   self.$watch(watchFn, function(newValue, oldValue) {
  //     newValues[i] = newValue;
  //     oldValues[i] = oldValue;
  //     if (!changeReactionScheduled) {
  //       changeReactionScheduled = true;
  //       self.$evalAsync(watchGroupListener);
  //     }
  //   });
  // });
};
```

`$watchGroup` 需要的最后一个功能是销毁 watcher。开发者销毁 watcher 组合的方法销毁单个 watcher 一样：那就是利用 `$watchGroup` 调用时返回的函数进行销毁。

_test/scope\_spec.js_

```js
it('can be deregistered', function() {
  var counter = 0;

  scope.aValue = 1;
  scope.anotherValue = 2;

  var destroyGroup = scope.$watchGroup([
    function(scope) { return scope.aValue; },
    function(scope) { return scope.anotherValue; }
  ], function(newValues, oldValues, scope) {
    counter++;
  });
  scope.$digest();

  scope.anotherValue = 3;
  destroyGroup();
  scope.$digest();

  expect(counter).toEqual(1);
});
```

这里我们测试在调用销毁函数后，后续对数据的更改是否还会再触发 listener 函数。

由于每个 watcher 调用后本来就会返回用于销毁该 watcher 的函数，我们要做的就是将它们收集起来，然后创建一个新的销毁函数，这个销毁函数会逐个调用每个 watch 的销毁函数：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  // var self = this;
  // var oldValues = new Array(watchFns.length);
  // var newValues = new Array(watchFns.length);
  // var changeReactionScheduled = false;
  // var firstRun = true;
  // if (watchFns.length === 0) {
  //   self.$evalAsync(function() {
  //     listenerFn(newValues, newValues, self);
  //   });
  //   return;
  // }

  // function watchGroupListener() {
  //   if (firstRun) {
  //     firstRun = false;
  //     listenerFn(newValues, newValues, self);
  //   } else {
  //     listenerFn(newValues, oldValues, self);
  //   }
  //   changeReactionScheduled = false;
  // }

  var destroyFunctions = _.map(watchFns, function(watchFn, i) {
    return self.$watch(watchFn, function(newValue, oldValue) {
      // newValues[i] = newValue;
      // oldValues[i] = oldValue;
      // if (!changeReactionScheduled) {
      //   changeReactionScheduled = true;
      //   self.$evalAsync(watchGroupListener);
      // }
    });
  });

  return function() {
    _.forEach(destroyFunctions, function(destroyFunction) {
      destroyFunction();
    });
  };
};
```

由于也可能会出现 watch 数组为空的情况，我们需要保证在这种情况下也会生成一个销毁 watcher 的函数。listener 在这种情况下只会被调用一次，但我们依然可以在第一次 digest 启动之前调用注销函数，在这种情况下，即使是一次调用也需要跳过：

_test/scope\_spec.js_

```js
it('does not call the zero-watch listener when deregistered first', function() {
  var counter = 0;

  var destroyGroup = scope.$watchGroup([], function(newValues, oldValues, scope) {
    counter++;
  });
  destroyGroup();
  scope.$digest();

  expect(counter).toEqual(0);
});
```

要处理这种情况，我们只需要在代码中加入一个布尔值标识，然后在调用 listener 函数之前检查一下这个布尔值标识的值就可以了：

_src/scope.js_

```js
Scope.prototype.$watchGroup = function(watchFns, listenerFn) {
  // var self = this;
  // var oldValues = new Array(watchFns.length);
  // var newValues = new Array(watchFns.length);
  // var changeReactionScheduled = false;
  // var firstRun = true;

  if (watchFns.length === 0) {
    var shouldCall = true;
    self.$evalAsync(function() {
      if (shouldCall) {
        listenerFn(newValues, newValues, self);
      }
    });
    return function() {
      shouldCall = false;
    };
  }

  // function watchGroupListener() {
  //   if (firstRun) {
  //     firstRun = false;
  //     listenerFn(newValues, newValues, self);
  //   } else {
  //     listenerFn(newValues, oldValues, self);
  //   }
  //   changeReactionScheduled = false;
  // }

  // var destroyFunctions = _.map(watchFns, function(watchFn, i) {
  //   return self.$watch(watchFn, function(newValue, oldValue) {
  //     newValues[i] = newValue;
  //     oldValues[i] = oldValue;
  //     if (!changeReactionScheduled) {
  //       changeReactionScheduled = true;
  //       self.$evalAsync(watchGroupListener);
  //     }
  //   });
  // });

  // return function() {
  //   _.forEach(destroyFunctions, function(destroyFunction) {
  //     destroyFunction();
  //   });
  // };
};
```




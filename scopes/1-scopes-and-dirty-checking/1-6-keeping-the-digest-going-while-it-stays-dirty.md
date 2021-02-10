### 还有脏值时继续运行 digest（Keeping The Digest Going While It Stays Dirty）

虽然我们已经实现了脏值检测系统的核心部分，但离开发完成还远着呢。比如下面这种典型场景，我们还没处理好：listener 函数改变了一个作用域属性，同时有另一个 watcher 在侦听该属性时，那实际上在当前 digest 运行过程中这个 watcher 时无法感知到这个属性值时发生了变化了的：

_test/scope\_spec.js_

```js
it('triggers chained watchers in the same digest', function() {
  scope.name = 'Jane';

  scope.$watch(
    function(scope) { return scope.nameUpper; },
    function(newValue, oldValue, scope) {
      if (newValue) {
        scope.initial = newValue.substring(0, 1) + '.';
      }
    }
  );

  scope.$watch(
    function(scope) { return scope.name; },
    function(newValue, oldValue, scope) {
      if (newValue) {
        scope.nameUpper = newValue.toUpperCase();
      }
    }
  );

  scope.$digest();
  expect(scope.initial).toBe('J.');

  scope.name = 'Bob';
  scope.$digest();
  expect(scope.initial).toBe('B.');
});
```

我们在作用域里面有两个 watcher：一个会侦听 `nameUpper` 属性，它会根据这个属性来生成 `initial` 属性，而另一个侦听的是 `name` 属性，并根据这个属性生成要赋给 `nameUpper` 属性的值。我们希望达到的效果是，在同一个 digest 阶段中，当作用域上的 `name` 发生变化时，`nameUpper` 和 `initial` 的属性值会相应发生改变。但目前的情况显然不是这样的。

> 这里我们故意先注册依赖另一个监听器返回值的 watcher。这是因为如果把注册顺序换回来，这时 watcher 注册执行的顺序正好就是依赖的先后顺序，测试就会直接通过了。但我们希望 watcher 之间的依赖关系不受注册顺序的影响。

我们需要修改 digest 的代码，让其在侦听属性值停止变化之前持续对所有 watcher 进行遍历。多轮检查是能让依赖其他侦听器的 watcher 感知到属性发生变化的唯一方法。

我们先把 `$digest` 函数重命名为 `$$digestOnce`，然后对它进行调整，让这个函数能把所有 watcher 都执行一次，最终返回一个标识本轮是否发生了变化的布尔值：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  var newValue, oldValue, dirty;
  _.forEach(this.$$watchers, function(watcher) {
    // newValue = watcher.watchFn(self);
    // oldValue = watcher.last;
    if (newValue !== oldValue) {
      // watcher.last = newValue;
      // watcher.listenerFn(newValue,
      //   (oldValue === initWatchVal ? newValue : oldValue),
      //   self);
      dirty = true;
    }
  });
  return dirty;
};
```

然后，我们需要重新定义 `$digest` 函数，它会执行一个“外部的循环”，只要发现值发生了变化就一直调用 `$$digestOnce`：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  var dirty;
  do {
    dirty = this.$$digestOnce();
  } while (dirty);
};
```

现在的 `$digest`  至少会对所有 watcher 进行一轮遍历。在第一轮遍历时，一旦发现有侦听值发生改变，那么整一轮会被标记为是“脏”（dirty）的，需要对所有 watcher 进行第二轮的遍历。这个过程会一直持续下去，直到本轮中没有侦听值发生变化，这时我们才认为 digest 彻底稳定下来了。

> 实际上，Angular 源码中并没有 `$$digestOnce` 函数。Angular 源码中把循环执行 digest 的代码逻辑都整合到 `$digest` 方法里面去了。我们追求的是清晰展示实现原理，而不是极致的性能表现，所以才把内部的循环单独剥离为一个函数。

现在我们可以观察到 Angular watch 函数的另一个特性：在每次 digest 完成之前，watch 函数可能已经被运行很多次了。这也是为什么我们希望 watcher 应当具有[幂等性](https://en.wikipedia.org/wiki/Idempotence)，也就是 watch 函数应该没有副作用或者副作用只会发生有限次的。举个例子，如果我们在 watch 函数里发起了一个 Ajax 请求，我们无法保证应用最终会发起多少次 Ajax 请求。


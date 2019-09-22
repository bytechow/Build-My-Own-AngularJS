### 尚有脏值存在时继续运行 digest（Keeping The Digest Going While It Stays Dirty）

虽然我们已经实现了脏值检测的核心了，但我们离完成还远着呢。举例来说，我们就还没能支持这样一个典型场景：listener 函数可能也会改变作用域上的属性。如果出现了这种情况，并且还有另一个 watcher 在侦听该属性时，那么在当前 digest 运行过程中它无法察觉到这个属性是发生了变化的：

_test/scope_spec.js_

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

我们在作用域里面有两个 watcher：一个会侦听 `nameUpper` 属性，它会根据这个属性来生成`initial`属性，而另一个侦听的是 `name` 属性，并根据这个属性生成赋给 `nameUpper` 属性的值。我们希望产生的效果是，在同一个 digest 阶段中，当作用域上的 `name` 发生变化时，`nameUpper` 和 `initial` 的属性值会相应发生改变。但目前的情况并不是这样的。

> 我们故意把注册 watcher 的顺序调成依赖的一方先注册。如果顺序调换过来，因为 watcher 是按照正常的依赖顺序排列的，这个测试直接就能通过了。但无论如何，我们希望出现的是 watcher 之间的依赖不受注册顺序的影响。

我们需要做的就是修改 digest 的代码，让其在侦听属性值停止变化之前持续对所有 watcher 进行遍历。多轮检查，是我们能让依赖其他 watcher 的 watcher 感知到属性变化的唯一方法。

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

然后，我们需要重新定义 `$digest`，让它能够运行“外面的循环”，只要发现还有变化就继续调用 `$$digestOnce`：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  var dirty;
  do {
    dirty = this.$$digestOnce();
  } while (dirty);
};
```

`$digest` 现在可以对所有的 watcher 进行多轮的执行了。如果在第一轮运行时，任何一个被侦听的值发生了改变，则这一轮会被标记为是“脏”的，会对所有 watcher 进行第二轮执行。这样的检查执行会继续循环执行，直到该轮中没有任何一个被侦听值发生变化时才会停止，这种停止变化状态被认为是稳定的状态。

> 实际上 Angular 作用域的代码中并没有一个叫 `$$digestOnce` 的函数。Angular 源码中把循环执行 digest 的代码都放在 `$digest` 方法里面了。我们追求的是清晰地展现实现原理而不是极致的性能体现，所以才把内部的循环单独剥离为一个函数。

现在，我们能看到关于 Angular watch 函数另一个需要注意的点：它们可能会在每次 digest 完成之前运行很多次。这也是为什么人们都说 watcher 应该具有[幂等性](https://en.wikipedia.org/wiki/Idempotence)的：也就是 watch 函数应该不包含副作用或者副作用只会发生有限次的。举个例子，如果在一个 watch 函数里会发起一个 Ajax 请求，我们无法保证应用最终会发起多少次请求。


### 调用 `$apply`，`$evalAsync` 和 `$applyAsync` 时会对整个树结构进行 digest

#### Digesting The Whole Tree from $apply, $evalAsync, and $applyAsync

正如我们在上面章节看到的那样，`$digest` 只会从当前作用域开始往下执行。但对于 `$apply` 来说，情况就不一样了。当我们在 Angular 调用 `$apply` 时，它会从整个作用域树结构的最顶端开始执行 digest。下面这个单元测试就说明了目前我们还没实现这样的需求：

_test/scope\_spec.js_

```js
it('digests from root on $apply', function() {
  var parent = new Scope();
  var child = parent.$new();
  var child2 = child.$new();

  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  child2.$apply(function(scope) {});

  expect(parent.counter).toBe(1);
});
```

从目前的测试结果可以看到，在（孙）子元素上调用 `$apply` 时，并没有触发它祖父元素上的 watcher。

要实现这个效果，我们先要在作用域上保存对根元素的引用，这样才能在根元素上触发 digest。当然，我们也可以沿着原型链一步步找到根作用域，但直接在作用域上保存一个 `$root` 属性会方便得多。我们可以在根作用域的构造函数上初始化这个变量：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  // this.$$applyAsyncQueue = [];
  // this.$$applyAsyncId = null;
  // this.$$postDigestQueue = [];
  this.$root = this;
  // this.$$children = [];
  // this.$$phase = null;
}
```

这样的话，我们就可以利用原型链继承的特性，让树结构里面所有的作用域就都能访问到这个 `$root` 变量了。

我们需要对 `$apply` 进行修改，但也很简单。我们不再调用当前作用域上的 `$digest` ，转而调用根作用域上的 `$digest` 即可：

_src/scope.js_

```js
Scope.prototype.$apply = function(expr) {
  try {
    this.$beginPhase('$apply');
    return this.$eval(expr);
  } finally {
    this.$clearPhase();
    this.$root.$digest();
  }
};
```

注意，这里的 `$eval` 方法依旧是在当前作用域上下文内执行的，而不是根作用域。我们只是希望 digest 能从根作用域一路运行下来而已。

实际上，`$apply` 从根作用域开始往下执行 digest 的这个特性，正是我们推荐使用它而不是更为简单 `$digest` 方法来集成外部代码的原因之一：如果我们不清楚哪些作用域与即将发生的变更有关，那我们完全可以对所有作用域进行脏值检测。

值得注意的是，由于 Angular 应用只有一个根作用域，调用 `$apply` 后确实会执行应用里所有作用域上的 watcher。再了解了 `$digest` 和 `$apply` 的区别以后，当你遇到提升性能的情况时，就可以用 `$digest` 来代替 `$apply` 了。

在介绍了 `$digest` 和 `$apply`（以及与 `$apply` 紧密关联的 `$applyAsync`）以后，我们还需要讨论另一个会触发 digest 的函数——`$evalAsync`。它的工作原理与 `$apply` 类似，是在根作用域上设定延迟执行的 digest 任务的，而不是在被调用的作用域上。用单元测试来说明下：

_test/scope\_spec.js_

```js
it('schedules a digest from root on $evalAsync', function(done) {
  var parent = new Scope();
  var child = parent.$new();
  var child2 = child.$new();

  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  child2.$evalAsync(function(scope) {});

  setTimeout(function() {
    expect(parent.counter).toBe(1);
    done();
  }, 50);
});
```

这个单元测试跟前一个很像：都是在子作用域上调用 `$evalAsync`，然后看祖父作用域上的 watcher 是否会被执行。

由于我们已经可以很方便地访问到根作用域，修改 `$evalAsync` 的工作就变得很简单了。同样地，我们只需要用根作用域来代替 `this` 调用 `$digest` 就可以了：

_src/scope.js_

```js
Scope.prototype.$evalAsync = function(expr) {
  // var self = this;
  // if (!self.$$phase && !self.$$asyncQueue.length) {
  //   setTimeout(function() {
  //     if (self.$$asyncQueue.length) {
        self.$root.$digest();
  //     }
  //   }, 0);
  // }
  // this.$$asyncQueue.push({ scope: this, expression: expr });
};
```

有了 `$root` 属性以后，我们还可以检查一下 digest 的代码，确保我们引用到的是正确的 `$$lastDirtyWatch`，以便检查短路优化的相关状态。无论我们现在是在哪个作用域上调用 `$digest`，`$$lastDirtyWatch` 指向的应该都是根作用域上的 `$$lastDirtyWatch`。

我们应该在 `$watch` 函数中把对 `$$lastDirtyWatch` 的引用改成使用 `$root.$$lastDirtyWatch`：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  // var self = this;
  // var watcher = {
  //   watchFn: watchFn,
  //   listenerFn: listenerFn || function() {},
  //   last: initWatchVal,
  //   valueEq: !!valueEq
  // };
  // this.$$watchers.unshift(watcher);
  // this.$root.$$lastDirtyWatch = null;
  return function() {
    // var index = self.$$watchers.indexOf(watcher);
    // if (index >= 0) {
    //   self.$$watchers.splice(index, 1);
      self.$root.$$lastDirtyWatch = null;
    // }
  };
};
```

同样地，在 `$digest` 中我们也要这样处理：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  this.$root.$$lastDirtyWatch = null;
  // this.$beginPhase('$digest');

  // if (this.$$applyAsyncId) {
  //   clearTimeout(this.$$applyAsyncId);
  //   this.$$flushApplyAsync();
  // }

  // do {
  //   while (this.$$asyncQueue.length) {
  //     try {
  //       var asyncTask = this.$$asyncQueue.shift();
  //       asyncTask.scope.$eval(asyncTask.expression);
  //     } catch (e) {
  //       console.error(e);
  //     }
  //   }
  //   dirty = this.$$digestOnce();
  //   if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty || this.$$asyncQueue.length);

  // this.$clearPhase();
  // while (this.$$postDigestQueue.length) {
  //   try {
  //     this.$$postDigestQueue.shift()();
  //   } catch (e) {
  //     console.error(e);
  //   }
  // }
};
```

最后是 `$$digestOnce`：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  // var dirty;
  // this.$$everyScope(function(scope) {
  //   var newValue, oldValue;
  //   _.forEachRight(scope.$$watchers, function(watcher) {
  //     try {
  //       if (watcher) {
  //         newValue = watcher.watchFn(scope);
  //         oldValue = watcher.last;
          if (!scope.$$areEqual(newValue, oldValue, watcher.valueEq)) {
            scope.$root.$$lastDirtyWatch = watcher;
            // watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
            // watcher.listenerFn(newValue,
            //   (oldValue === initWatchVal ? newValue : oldValue),
            //   scope);
            // dirty = true;
          } else if (scope.$root.$$lastDirtyWatch === watcher) {
            // dirty = false;
            // return false;
          }
  //       }
  //     } catch (e) {
  //       console.error(e);
  //     }
  //   });
  //   return dirty !== false;
  // });
  // return dirty;
};
```




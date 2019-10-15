### 调用 `$apply`，`$evalAsync` 和 `$applyAsync` 时会对整个树结构进行 digest
#### Digesting The Whole Tree from $apply, $evalAsync, and $applyAsync

正如我们在上面的章节中看到的那样，`$digest` 只会从当前作用域开始往下执行。但对于 `$apply` 来说，情况就不一样了。当我们在 Angular 中调用 `$apply`，它会从整个作用域树结构的最顶端开始执行 digest。下面这个单元测试就能说明目前我们还没实现这样的效果：

_test/scope_spec.js_

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

我们能看到，当我们在（孙）子元素上调用 `$apply` 时，并未能触发它祖父元素上的 watcher。

要实现这个效果，我们首先需要在作用域上保存它们的根元素的引用，这样才能在根元素上触发 digest。我们也可以沿着原型链找到根作用域，但显式地在作用域上保存一个 `$root` 属性会直接得多。我们可以在根作用域的构造函数上设置这个变量：

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

这样的话，利用原型继承链的特性，树结构里面所有的作用域就都能访问到这个 `$root` 变量了。

我们仍然需要在 `$apply` 中进行修改，但也非常简单。我们在根作用域上调用 `$digest`，而不是在当前作用域上调用：

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

注意，这里我们还是会在当前作用域的语境下运行传入的函数，而不是在根作用域上，通过调用 `this` 上的 `$eval` 方法。我们只是希望 digest 能从根作用域开始一直运行下来而已。

事实上，`$apply` 会从根作用域开始往下执行 digest 的特性，正是我们会推荐使用它而不是更为简单 `$digest` 方法来集成外部代码的原因之一：如果你不太清楚哪些作用域会与将要发生的更改相关联，那完全可以把它们都包含进来。

值得注意的是，由于 Angular 应用只有一个根作用域，`$apply` 确实会导致整个应用里的所有作用域上的 watcher 都会被执行。知道了 `$digest` 和 `$apply` 的区别之后，当你遇到需要额外性能的情况时，你就可以用 `$digest` 来代替 `$apply` 了。

在介绍了 `$digest` 和 `$apply`（以及与 `$apply` 紧密关联的 `$applyAsync`）之后，我们还需要讨论另一个会触发 digest 的函数，即 `$evalAsync`。它的工作原理与 `$apply` 类似，是在根作用域上设定延时的 digest 任务的，而不是在被调用的作用域上。用单元测试表示就是：

_test/scope_spec.js_

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

这个单元测试跟之前那一个很像：我们在子作用域上调用 `$evalAsync`，看祖父作用域上的 watcher 是否会被执行。

由于我们已经可以轻易地访问到根作用域，要修改 `$evalAsync` 就变得很简单了。我们只需要用根作用域而不是 `this` 来调用 `$digest` 就可以了：

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

有了 `$root` 属性的帮助，我们还可以回顾一下 digest 的代码，保证我们引用的是正确的 `$$lastDirtyWatch` 以便检查短路优化的相关状态。无论我们现在是在哪个作用域上调用 `$digest`，`$$lastDirtyWatch` 指向的应该一直都是根作用域的 `$$lastDirtyWatch`。

我们应该在 `$watch` 函数改成引用 `$root.$$lastDirtyWatch`：

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

在 `$digest` 中我们也要这样处理：

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
### 孤立的作用域
#### Isolated Scopes

我们已经看到了父子作用域在原型继承链上的联系是多么的紧密。无论父作用域上有什么属性，子作用域都可以访问到。如果这个属性是一个对象或者数组，子作用域还可以修改它的内容。

但有时候我们并不需要这么紧密的联系。有时，如果能有一个作用域本身是作用域树的一部分但没有权限访问父作用域数据的话，是很方便的。这就是所谓的_孤立的作用域_（isolated scope）。

孤立作用域背后的思想是很简单的：我们会像之前一样创建一个作用域，它会作为作用域树的一部分，但我们不会让它在原型链上对它的父作用域进行继承。它会从父作用域的原型链结构中切割出来，或者说是被孤立起来。

我们可以通过传递一个布尔值参数给 `$new` 函数来创建一个孤立作用域。当这个参数值为 `true` 时，创建的作用域就会是孤立的。当只为 `false`（或者是被忽略了，或者是 `undefined`），就会使用原型继承。当作用域是孤立的，它就无法访问到父作用域上的数据：

_test/scope_spec.js_

```js
it('does not have access to parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';
  
  expect(child.aValue).toBeUndefined();
});
```

而且由于我们已经无法访问到父作用域上的属性，我们自然就没办法对这些属性进行侦听了：

```js
it('cannot watch parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  
  child.$digest();
  expect(child.aValueWas).toBeUndefined();
});
```

我们会在 `$new` 函数中设置孤立的作用域。根据传入的布尔值属性来决定是创建跟以往一样的子作用域，还是使用 `Scope` 构造函数来创建一个独立的作用域。这两种方式创建的新作用域都会被添加到当前作用域的子作用域数组（children）中去：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

> 如果你在 Angular 指令中用过孤立作用域，你会知道其实孤立作用域通常不会跟它的父作用域完全割裂开来的。相反，我们会明确定义出需要从父作用域中获取的属性映射。
> 
> 然而，作用域中并没有加入这个机制。这个机制是指令功能的一部分。后面我们讲到指令作用域链接（directive scope linking）的时候会再来讨论这个知识点。

由于现在破坏了原型继承链，我们需要对本章先前介绍过的 `$digest`、`$apply`、`$evalAsync` 和 `$applyAsync` 进行一次回顾。

首先，我们希望 `$digest` 可以对整个继承关系树进行遍历。由于我们已经把孤立作用域放到父作用域的 `$$children` 中，所以这个问题已经被解决了。这也意味着下面这个测试已经可以通过了：

_test/scope_spec.js_

```js
it('digests its isolated children', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  child.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );

  parent.$digest();
  expect(child.aValueWas).toBe('abc');
});
```

但我们还没有处理 `$apply`、`$evalAsync` 和 `$applyAsync` 三种情况。我们假设这些函数都会从根作用域开始往下进行 digest 的过程，但位于层次结构中的孤立作用域会破坏这个假设，如下面两个未能通过的单元测试所示：

```js
it('digests from root on $apply when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);
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

it('schedules a digest from root on $evalAsync when isolated', function(done) {
  var parent = new Scope();
  var child = parent.$new(true);
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

> 由于 `$applyAsync` 是基于 `$apply` 实现的，所以它们俩出现的问题是一样，只要我们修复好 `$apply` 上的问题，`$applyAsync` 自然也没问题了。

注意，这两个测试其实跟之前开发 `$apply` 和 `$evalAsync` 设定的测试基本相同，只是这里我们把其中一个作用域变成孤立的而已。

测试没有通过的原因在于我们现在是依然是依赖 `$root` 属性来指向树结构的跟作用域。没有被孤立的作用域能通过继承机制从实际的根元素访问到这个属性，但孤立作用域不能。实际上，由于我们使用 `Scope` 构造函数来创建孤立作用域，而这个构造函数本身会指派一个 `$root` 属性，结果就是每一个孤立作用域都有一个指向自身的 `$root` 属性。这并不是我们想要的。

要修复这个问题也很简单，我们只需要在 `$new` 中把新创建的孤立作用域的 `$root` 属性重新指向真正的根作用域即可：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  // var child;
  if (isolated) {
  //   child = new Scope();
    child.$root = this.$root;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

在了解有关继承的所有内容之前，我们还需要在孤立作用域这一节中修复一个问题，这个问题与 `$evalAsync`、`$applyAsync` 和 `$$postDigest` 函数中存储的队列有关。回想一下，我们是在 `$digest` 函数中处理 `$$asyncQueue` 和 `$$postDigestQueue`，而在 `$$flushApplyAsync` 中处理 `$$applyAsyncQueue`。在这两个地方，我们都没有对子作用域或父作用域进行额外的处理。我们只是简单地假设每个队列只有一个实例，它指向的是整个树结构中所有的异步任务队列。

对于非孤立的作用域来说，情况是：每当我们在任意一个作用域中访问一个队列时，访问到的队列始终是一样的，因为每个作用域都继承了这个队列。但对孤立作用域来说就不一样了。像前面提到的 `$root`，孤立作用域创建了本地版本的 `$asyncQueue`、`$applyAsyncQueue` 和 `$$postDigestQueue` ，会屏蔽根作用域上三个同名属性。这就导致了一个不良的后果，在孤立作用域上使用 `$evalAsync` 或者 `$$postDigest` 设定的延时函数永远无法执行：

_test/scope_spec.js_

```js
it('executes $evalAsync functions on isolated scopes', function(done) {
  var parent = new Scope();
  var child = parent.$new(true);

  child.$evalAsync(function(scope) {
    scope.didEvalAsync = true;
  });
  
  setTimeout(function() {
    expect(child.didEvalAsync).toBe(true);
    done();
  }, 50);
});

it('executes $$postDigest functions on isolated scopes', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  child.$$postDigest(function() {
    child.didPostDigest = true;
  });
  parent.$digest();

  expect(child.didPostDigest).toBe(true);
});
```

跟 `$root` 一样，无论是不是孤立作用域，我们希望这个作用域使用的 `$$asyncQueue` 和 `$$postDigestQueue` 指向同一个引用位置。如果作用域不是孤立作用域，它会自动获得正确的引用位置。如果是孤立作用域，我们就需要显式地进行赋值了：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  // var child;
  if (isolated) {
    // child = new Scope();
    // child.$root = this.$root;
    child.$$asyncQueue = this.$$asyncQueue;
    child.$$postDigestQueue = this.$$postDigestQueue;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  // }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

`$$applyAsyncQueue` 的情况就有点不一样了：由于任务队列的处理是由 `$$applyAsyncId` 来控制的，而现在树结构中的作用域可能会有自己的 `$$applyAsyncId` 实例，我们实际上会有几个 `$applyAsync` 进程，每个孤立作用域一个。这就违背了 `$applyAsync` 的合并 `$apply` 调用的初衷了。

我们可以利用 `$applyAsyncQueue` 会在 `$digest` 函数中处理完的事实来进行测试。如果我们在子作用域调用 `$digest`，父作用域设定的 `$applyAsync` 任务就应该会在这个 digest 中进行处理，但目前并不是这样的：

_test/scope_spec.js_

```js
it("executes $applyAsync functions on isolated scopes", function() {
  var parent = new Scope();
  var child = parent.$new(true);
  var applied = false;

  parent.$applyAsync(function() {
    applied = true;
  });
  child.$digest();
  
  expect(applied).toBe(true);
});
```

首先，就像 `$evalAsync` 和 `$$postDigest` 的任务队列一样，我们需要在作用域之间共享这个队列：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  // var child;
  if (isolated) {
    // child = new Scope();
    // child.$root = this.$root;
    // child.$$asyncQueue = this.$$asyncQueue;
    // child.$$postDigestQueue = this.$$postDigestQueue;
    child.$$applyAsyncQueue = this.$$applyAsyncQueue;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

其次，我们需要共享 `$$applyAsyncId` 属性。我们不能直接在 `$new` 拷贝这个属性，因为孤立作用域还是需要能定义自己的 `$$applyAsyncId` 属性的。但我们可以显式地 `$root`：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$root.$$lastDirtyWatch = null;
  // this.$beginPhase('$digest');

  if (this.$root.$$applyAsyncId) {
    clearTimeout(this.$root.$$applyAsyncId);
    // this.$$flushApplyAsync();
  }

  // do {
  //   while (this.$$asyncQueue.length) {
  //     try {
  //       var asyncTask = this.$$asyncQueue.shift();
  //       asyncTask.scope.$eval(asyncTask.expression);
  //     } catch (e) {
  //       console.error(e);

  //     }
  //     dirty = this.$$digestOnce();
  //     if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
  //       throw '10 digest iterations reached';
  //     }
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

Scope.prototype.$applyAsync = function(expr) {
  // var self = this;
  // self.$$applyAsyncQueue.push(function() {
  //   self.$eval(expr);
  // });
  if (self.$root.$$applyAsyncId === null) {
    self.$root.$$applyAsyncId = setTimeout(function() {
      // self.$apply(_.bind(self.$$flushApplyAsync, self));
    }, 0);
  }
};

Scope.prototype.$$flushApplyAsync = function() {
  // while (this.$$applyAsyncQueue.length) {
  //   try {
  //     this.$$applyAsyncQueue.shift()();
  //   } catch (e) {
  //     console.error(e);
  //   }
  // }
  this.$root.$$applyAsyncId = null;
};
```

最终，我们把所需的一切都准备好了！
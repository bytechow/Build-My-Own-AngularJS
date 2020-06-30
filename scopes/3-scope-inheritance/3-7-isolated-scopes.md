### 隔离作用域
#### Isolated Scopes

我们已经看到了在原型继承中父子作用域之间的联系非常密切。无论父作用域上有什么属性，子作用域都可以访问到。如果这个属性是一个对象或者数组，子作用域还可以修改它的内容。

但有时我们并不希望它们之前的联系太过紧密，如果子作用域本身作为作用域树的一部分但没有权限访问父作用域数据的话会更方便。这就是所谓的_隔离作用域_（isolated scope）。

隔离作用域的核心概念是很简单的：我们会像之前一样创建一个作用域，它会成为作用域树的一部分，但我们不会让它在原型链继承它的父作用域。它会从父作用域的原型链结构中切割出来，或者说被隔离起来。

我们可以通过给 `$new` 传入一个布尔值参数来创建一个隔离作用域。当这个参数的值为 `true` 时，创建的作用域就会是被隔离了的。相反，当值为 `false`（`undefined` 或被忽略）时，就会使用原型继承。当作用域是隔离时，它就无法访问到父作用域上的数据：

_test/scope_spec.js_

```js
it('does not have access to parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';
  
  expect(child.aValue).toBeUndefined();
});
```

既然无法访问到父作用域上的属性，自然也没办法对这些属性进行侦听了：

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

隔离作用域是在 `$new` 方法中创建的。我们需要根据传入的布尔值属性来决定是创建跟以往一样的子作用域，还是使用 `Scope` 构造函数来创建一个隔离作用域。这两种方式创建的新作用域都会被添加到当前作用域的子作用域数组（children）中去：

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

> 如果你在 Angular 指令中用过隔离作用域，就会知道其实隔离作用域一般不会完全与自己的父作用域割裂开来的。相反，我们会根据需要从父作用域中获取的数据，明确定义出一个属性映射。
> 
> 但目前我们的作用域还没有实现这个机制。这个机制是指令功能的一部分。后面我们讲到指令作用域的链接（directive scope linking）时再来详细介绍。

既然打破了原型继承链，我们需要重新回顾一下本章讨论过的 `$digest`、`$apply`、`$evalAsync` 和 `$applyAsync`。

首先，我们需要 `$digest` 可以遍历整个继承关系树。由于前面我们已经把隔离作用域放到父作用域下的 `$$children` 属性中，所以这个问题已经解决了，同时下面的这个单元测试也应该通过了：

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

下面还需要处理 `$apply`、`$evalAsync` 和 `$applyAsync` 三个方法。我们希望调用这些方法时，都是从根作用域开始往下进行 digest，但在树层次结构中的隔离作用域会打破了这个假设，下面两个失败的单元测试说明了这一点：

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

> 由于 `$applyAsync` 是在 `$apply` 的基础上实现的，它俩遇到的问题也是一样，只要我们修复了 `$apply`，`$applyAsync` 自然也没问题了。

注意，这两个单元测试其实跟之前开发 `$apply` 和 `$evalAsync` 时所写的两个用力相同，只是在本例中我们将一个作用域换成是隔离作用域而已。

这两个测试没有通过，原因在于我们现在依然是依赖 `$root` 属性指向根作用域。普通的作用域可以通过继承机制访问到根元素上的这个属性，但隔离作用域不能。实际上，由于我们会使用 `Scope` 构造函数来创建隔离作用域，这个构造函数本身会初始化一个 `$root` 属性，结果每一个孤立作用域都会有一个指向自身的 `$root` 属性。这就不是我们想要的了。

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

在了解与继承相关的内容之前，我们还需要在隔离作用域的上下文中修复一个问题，这个问题与 `$evalAsync`、`$applyAsync` 和 `$$postDigest` 函数中存储的队列有关。回想一下，我们是在 `$digest` 函数处理 `$$asyncQueue` 和 `$$postDigestQueue` 两个队列，然后在 `$$flushApplyAsync` 中处理 `$$applyAsyncQueue`。在这两个地方，我们都没有对子作用域或父作用域进行额外的处理。我们只是简单地假设每个队列都有一个实例，它指向的是整个树结构中所有排队的任务。

对于非隔离作用域来说，情况会是这样的：当我们访问任意作用域上的一个队列时，访问到的队列都是同一个，因为这个队列会被每一个作用域继承。但对隔离作用域来说情况就不一样了。与前面提到的 `$root` 类似，隔离作用域创建的 `$asyncQueue`、`$applyAsyncQueue` 和 `$$postDigestQueue` 方法会屏蔽掉根作用域上的三个同名队列。这就产生了一个不幸的影响，即在隔离作用域上使用 `$evalAsync` 或者 `$$postDigest` 设定的延时函数永远不会被执行：

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

跟 `$root` 一样，无论作用域是不是隔离作用域，我们都希望所有作用域共享 `$$asyncQueue` 和 `$$postDigestQueue` 的同一副本。如果作用域不是隔离作用域，它能自动得到一个副本。但如果是隔离作用域，我们就需要显式地进行赋值了：

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

`$$applyAsyncQueue` 出现的问题就不太一样了：`$$applyAsyncQueue`  是否执行任务队列是由 `$$applyAsyncId` 控制的，而现在树结构中的作用域可能会有自己的 `$$applyAsyncId` 实例，所以实际上会出现几个 `$applyAsync` 进程，每个隔离作用域一个。这就违背了 `$applyAsync` 合并 `$apply` 调用的初衷了。

我们可以利用 `$applyAsyncQueue` 会在 `$digest` 函数中全部执行完毕的这个特性进行测试。如果我们在子作用域调用 `$digest`，父作用域设定的 `$applyAsync` 任务应该会被执行，但目前并不是这样的：

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

首先，像 `$evalAsync` 和 `$$postDigest` 任务队列一样，我们需要每个作用域都能访问到这个队列：

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

其次，我们需要共享 `$$applyAsyncId` 属性。但我们不能直接在 `$new` 拷贝这个属性，因为隔离作用域还要能对 `$$applyAsyncId` 属性进行赋值。但我们可以直接从 `$root` 获取到它的值：

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

这样隔离作用域的相关内容就都能正常运行了！
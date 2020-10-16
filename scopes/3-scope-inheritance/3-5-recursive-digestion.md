### 递归的 digest

#### Recursive Digestion

上一节讲到在作用域上调用 `$digest` 时，不能执行位于当前作用域层次之上的 watcher。但我们需要运行当前作用域以下的，也就是它子作用域上的 watcher。这是因为子作用域可能会通过原型链对父作用域上的属性进行侦听。

由于现在已经把原来的 watcher 数组拆分到各个作用域中 ，现在在父作用域上调用 `$digest` 时，并不会触发子作用域的脏值检查。我们需要修改 `$digest` 方法，使它不仅会在当前作用域上运行，还会在子作用域上运行。

我们遇到的第一个问题是当前作用域并不知道它有没有子作用域，或者这些子作用域是属于谁的。为此，我们需要在作用域对象中记录它的子作用域信息。无论是根作用域还是子作用域都需要进行记录。我们把子作用域信息都保存在 `$$children` 数组属性中：

_test/scope\_spec.js_

```js
it('keeps a record of its children', function() {
  var parent = new Scope();
  var child1 = parent.$new();
  var child2 = parent.$new();
  var child2_1 = child2.$new();

  expect(parent.$$children.length).toBe(2);
  expect(parent.$$children[0]).toBe(child1);
  expect(parent.$$children[1]).toBe(child2);

  expect(child1.$$children.length).toBe(0);

  expect(child2.$$children.length).toBe(1);
  expect(child2.$$children[0]).toBe(child2_1);
});
```

我们需要在根作用域的构造函数里初始化 `$$children`：

_src/scope.js_

```js
function Scope() {
  // this.$$watchers = [];
  // this.$$lastDirtyWatch = null;
  // this.$$asyncQueue = [];
  // this.$$applyAsyncQueue = [];
  // this.$$applyAsyncId = null;
  // this.$$postDigestQueue = [];
  this.$$children = [];
  // this.$$phase = null;
}
```

然后在创建它的子作用域时，把这个子作用域放入到这个数组中。同时子作用域也需要创建自己的 `$children` 数组（它会屏蔽父作用域上的同名属性），这样我们才不会遇到跟 `$$watchers` 一样的问题。这些操作都会被放在 `$new` 函数中：

```js
Scope.prototype.$new = function() {
  // var ChildScope = function() { };
  // ChildScope.prototype = this;
  // var child = new ChildScope();
  this.$$children.push(child);
  // child.$$watchers = [];
  child.$$children = [];
  // return child;
};
```

现在既然保存好了子作用域，我们就可以启用它们的 digest 了。我们希望父作用域执行 `$digest` 时可以一并启动子作用域上的 watcher：

_test/scope\_spec.js_

```js
it('digests its children', function() {
  var parent = new Scope();
  var child = parent.$new();

  parent.aValue = 'abc';
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

注意，这个测试基本上就是上一节单元测试的一个“镜像”版本，在上节的单元测试中我们断言子作用域上的 `$digest` 启动时不会运行父作用域上的 watcher。

要实现这一点，我们需要修改 `$$digestOnce`，让它能把整个树结构中的所有 watcher 都执行一遍。为了简化这个流程，我们会使用一个帮助函数 `$$everyScope`（灵感来源于 JavaScript 的 `Array.every` 方法），它能让整个树结构中的每个作用域都执行一个任意函数直到该函数的返回值是一个假值（falsy value）：

_src/scope.js_

```js
Scope.prototype.$$everyScope = function(fn) {
  if (fn(this)) {
    return this.$$children.every(function(child) {
      return child.$$everyScope(fn);
    });
  } else {
    return false;
  }
};
```

这个函数会在当前作用域调用一次 `fn` 函数，然后在每一个子作用域上递归调用此函数：

现在我们就可以使用这个函数来包裹 `$$digestOnce` 的处理逻辑，形成最外层的循环：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  var dirty;
  var continueLoop = true;
  // var self = this;
  this.$$everyScope(function(scope) {
    var newValue, oldValue;
    _.forEachRight(scope.$$watchers, function(watcher) {
      // try {
      //   if (watcher) {
          newValue = watcher.watchFn(scope);
          // oldValue = watcher.last;
          if (!scope.$$areEqual(newValue, oldValue, watcher.valueEq)) {
            // self.$$lastDirtyWatch = watcher;
            // watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
            // watcher.listenerFn(newValue,
            //   (oldValue === initWatchVal ? newValue : oldValue),
              scope);
          //   dirty = true;
          // } else if (self.$$lastDirtyWatch === watcher) {
            continueLoop = false;
            // return false;
    //       }
    //     }
    //   } catch (e) {
    //     console.error(e);
    //   }
    // });
    return continueLoop;
  });
  // return dirty;
};
```

现在 `$$digestOnce` 会遍历整个树结构，最后返回一个布尔值标识，这个标识表示树结构中是否有 watcher 变“脏”。

内部的循环会对作用域所在的树结构进行遍历，直到所有作用域都已经被访问过或者遇到了适用短路优化的情况。我们会使用 `continueLoop` 变量来记录当前是否满足短路优化的条件。如果这个变量变成 `false`，我们就同时跳出循环和 `$$digestOnce` 函数。

注意，在内层循环中

注意，我们使用的 `$$lastDirtyWatch` 属性总是指向最顶层的那个作用域。短路优化需要考虑作用域所在树结构范围内的所有 watcher。如果我们在当前作用域设置 `$$lastDirtyWatch` 属性，就会屏蔽父作用域上的同名属性。

> 实际上，AngularJS 的作用域上并没有 `$$children` 属性。如果查看源码，你能发现它把子作用域放到一个定制的、链表形式的变量组中：`$$nextSibling`，`$$prevSibling`，`$$childHead` 和 `$$childTail`。这是一种优化手段，无需进行常规的数组操作，同时可以降低增删元素的成本。跟使用 `$$children` 数组实现的效果一样。




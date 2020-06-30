### 脏值检测（Checking for Dirty Values）

如上所述，侦听器（watcher）中的 watch 函数应该要返回要侦听的数据。通常这个数据是来源于作用域（scope）的。为了更方便地访问作用域，我们可以直接把作用域作为参数传入到 watch 函数中。这样, watch 函数要访问或者返回作用域中的数据就很简便了：

```js
function(scope) {
  return scope.firstName;
}
```

这就是 watch 函数的一般形式：从作用域拿到一个值，然后返回这个值。

下面我们来增加一个单元测试，看作用域是否作为一个参数传入到了 watch 函数中：

_test/scope\_spec.js_

```js
it('calls the watch function with the scope as the argument', function() {
  var watchFn = jasmine.createSpy();
  var listenerFn = function() { };
  scope.$watch(watchFn, listenerFn);

  scope.$digest();

  expect(watchFn).toHaveBeenCalledWith(scope);
});
```

我们将一个 spy 函数作为 watch 函数，通过这个 spy 函数，我们可以检查函数调用的情况。

要让这个测试通过，最简单的方式是让 `$digest` 这样处理：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  var self = this;
  _.forEach(this.$$watchers, function(watcher) {
    watcher.watchFn(self);
    watcher.listenerFn();
  });
};
```

> 在本书中，我们会经常使用 `var self = this;` 来获取当前运行环境的 `this`。[A List Apart article](https://alistapart.com/article/getoutbindingsituations/) 这篇文章详细描述这种模式和它要解决的问题。

当然，这还不能符合我们的要求。我们希望 `$digest` 调用 watch 函数时，把它的返回值与这个函数上一次的返回值进行比较，看是否相等。如果返回值发生了改变，那就说明 watcher 就是变“脏” 了，这时才需要调用它的 listener 函数。我们再写一个测试用例：

_test/scope\_spec.js_

```js
it('calls the listener function when the watched value changes', function() {
  scope.someValue = 'a';
  scope.counter = 0;

  scope.$watch(
    function(scope) { return scope.someValue; },
    function(newValue, oldValue, scope) { scope.counter++; }
  );

  expect(scope.counter).toBe(0);

  scope.$digest();
  expect(scope.counter).toBe(1);

  scope.$digest();
  expect(scope.counter).toBe(1);

  scope.someValue = 'b';
  expect(scope.counter).toBe(1);

  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

首先，我们在作用域上添加了两个属性：一个是字符串，一个是数字类型的计数器）。然后添加了一个 watcher，这个 watcher 会对字符串类型的属性进行侦听，如果字符串发生了改变，计数器会自增 1。我们希望达到的效果是，当第一次运行 `$digest` 时，计数器自增 1，之后每次 `$digest` 时如果发现值发生了改变，计数器也会自增 1。

注意，在这个单元测试中，我们同时确定了 listener 的函数签名：它跟 watch 函数一样，会把作用域作为参数。而 watcher 的旧值和新值也会作为其余的两个参数传入到 listener 函数中，这样更方便应用开发者检查到底是什么发生了变化。

要实现这个效果，`$digest` 就需要记忆上一次 watch 函数的返回结果。之前我们已经为每一个 watcher 创建了一个对象，所以我们直接把这个旧值放到这个对象里面去就好了。下面是对 `$digest` 的最新版本，它会检查每个 watch 函数的值是否发生了变化：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var self = this;
  var newValue, oldValue;
  // _.forEach(this.$$watchers, function(watcher) {
    newValue = watcher.watchFn(self);
    oldValue = watcher.last;
    if (newValue !== oldValue) {
      watcher.last = newValue;
      watcher.listenerFn(newValue, oldValue, self);
    }
  // }); 
};
```

便利 watcher 时，我们会把 watch 函数的返回值与之前保存的 `last` 属性值进行比较。如果两个值不同，我们就调用 listener 函数，同时传入新值、旧值和作用域对象作为 listener 函数的参数。最后，我们会把新值赋值给 watcher 的 `last` 属性，以便之后再次进行比较。

现在，我们已经实现了 Angular 作用域的核心部分：添加 watcher 然后在 digest 中执行。

这里也能看出关于 Angular 作用域性能的一些重要特性：

* 单纯把数据添加到作用域上（这种行为本身）是不会对性能造成影响。如果属性没有被 watcher 侦听，那它是否在作用域上并不重要。Angular 不会对作用域上的所有属性进行遍历，仅会对 watcher 进行遍历。
* 每次执行 `$digest` 时都会把绑定在作用域上的 watch 函数逐个执行一次。因此，我们最好留意一下注册侦听器的数量，还要留意每一个 watch 函数或表达式的性能。




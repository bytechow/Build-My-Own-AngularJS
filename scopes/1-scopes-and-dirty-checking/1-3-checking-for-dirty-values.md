### 脏值检测（Checking for Dirty Values）

如上所述，一个 watcher 的 watch 函数应该要返回我们需要观察其变化的数据。通常这个数据是来源于作用域的。为了更方便地访问作用域，我们可以直接把作用域作为参数传入到 watch 函数中。这样, watch 函数访问或者返回作用域中的数据就变得很简便了：

```js
function(scope) {
  return scope.firstName;
}
```

这就是 watch 函数的一般形式了：从作用域中获取值并返回这个值。

下面我们增加一个单元测试，看作用域是否作为一个参数传入到了 watch 函数中：

_test/scope_spec.js_

```js
it('calls the watch function with the scope as the argument', function() {
  var watchFn    = jasmine.createSpy();
  var listenerFn = function() { };
  scope.$watch(watchFn, listenerFn);
 
  scope.$digest();

  expect(watchFn).toHaveBeenCalledWith(scope);
});
```

这次我们会把一个 spy 作为 watch 函数，通过这个 spy 来检查的调用情况。

要让这个测试通过，最简单的方式是让 `$digest` 这样做：

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

> 在本书中，我们会一直利用 `var self = this;` 的方式来获取当前运行环境的 `this`。[A List Apart article](https://alistapart.com/article/getoutbindingsituations/) 详细描述这种模式和它要解决的问题。

当然，这也并不是我们真正想要的。`$digest` 函数真正要做的工作是调用 watch 函数，看看它的返回值与这个函数上一次的返回值是否相等。如果返回值发生了改变，watcher 就是变 _脏_ 了，watcher 需要调用它的 listener 函数。下面我们来为此增加一个测试用例：

_test/scope_spec.js_

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
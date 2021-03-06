### 处理重复代码
#### Dealing with Duplication

上一节，我们定义了两个几乎相同的单元测试和两个完全相同的函数。显然，两种事件传播方式之间会有很多相似点。在继续开发之前，我们先解决代码重复的问题，这样就不用什么东西都要写两遍了。

我们可以事件分发这个公共行为提取到另一个函数中，然后让 `$emit` 和 `$broadcast` 都调用它就可以了。我们把这个函数命名为 `$$fireEventOnScope`：

_src/scope.js_

```js
Scope.prototype.$emit = function(eventName) {
  this.$$fireEventOnScope(eventName);
};

Scope.prototype.$broadcast = function(eventName) {
  this.$$fireEventOnScope(eventName);
};

Scope.prototype.$$fireEventOnScope = function(eventName) {
  var listeners = this.$$listeners[eventName] || [];
  _.forEach(listeners, function(listener) {
    listener();
  });
};
```

> 原本的 AngularJS 并没有 `$$fireEventOnScope` 函数，确实就是在 `$emit` 和 `$broadcast` 中写重复的代码来处理这个公共行为。

这样好多了。但我们还可以更进一步，把测试套件中的重复代码也消灭掉。我们可以把描述两个方法相同功能的测试用例放到一个循环中，`$emit` 的执行一次，`$broadcast` 的执行一次。在循环体里面，我们可以动态生成对应的测试函数。下面我们来用这种方式来改写前面的两个测试用例：

_test/scope_spec.js_

```js
_.forEach(['$emit', '$broadcast'], function(method) {

  it('calls listeners registered for matching events on ' + method, function() {
    var listener1 = jasmine.createSpy();
    var listener2 = jasmine.createSpy();
    scope.$on('someEvent', listener1);
    scope.$on('someOtherEvent', listener2);

    scope[method]('someEvent');
    
    expect(listener1).toHaveBeenCalled();
    expect(listener2).not.toHaveBeenCalled();
  });
});
```

我们之所以可以这样做，是因为 Jasmine 的 `describe` 代码块实际上是一个函数，我们可以在里面执行任意的代码。这样就能高效地生成两个 `it` 代码块，也就是我们需要的两个测试用例了。
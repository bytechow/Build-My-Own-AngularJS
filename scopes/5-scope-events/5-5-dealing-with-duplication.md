### 处理重复代码
#### Dealing with Duplication

我们定义了两个几乎相同的单元测试和两个完全相同的函数。很明显这这两种事件传播的方式会有很多相似点。在继续开发之前，我们先把这个重复代码的问题解决了，这样我们就不用写重复的代码了。

我们可以抽取交付事件的逻辑到一个函数中，这个函数能同时被 `$emit` 和 `$broadcast` 使用。我们把这个函数命名为 `$$fireEventOnScope`：

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

> AngularJS 源码中并没有 `$$fireEventOnScope` 函数，而仅仅是对 `$emit` 和 `$broadcast` 都会用到代码进行了复制。

这样好多了，但我们还可以再进一步，把测试套件中的重复代码也消灭掉。我们可以把测试用例中描述相同功能的代码放到一个循环中，一次运行 `$emit`，一次运行 `$broadcast`。在循环体里面，我们可以动态地找到对应的函数。下面我们来使用这种方法替换之前加入的两个测试用例：

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

我们可以这样做，是因为 Jasmine 的 `describe` 代码块实际上就是一个函数，我们可以在里面执行任意的代码。这个循环就可以生成两个 `it` 代码块作为我们需要的测试用例了。
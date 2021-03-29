### 禁用已销毁作用域上的 listener

#### Disabling Listeners On Destroyed Scopes

除了广播 `$destroy` 事件，销毁作用域的另一个副作用是让事件监听器不会再被触发：

_test/scope_spec.js_

```js
it('no longers calls listeners after destroyed', function() {
  var listener = jasmine.createSpy();
  scope.$on('myEvent', listener);

  scope.$destroy();
  
  scope.$emit('myEvent');
  expect(listener).not.toHaveBeenCalled();
});
```

我们可以重新把 `$$listeners` 重置为一个空对象，这样就能有效地丢弃掉所有的事件监听器：

```js
Scope.prototype.$destroy = function() {
  this.$broadcast('$destroy');
  if (this.$parent) {
    var siblings = this.$parent.$$children;
    var indexOfThis = siblings.indexOf(this);
    if (indexOfThis >= 0) {
      siblings.splice(indexOfThis, 1);
    }
  }
  this.$$watchers = null;
  this.$$listeners = {};
};
```

这样处理后，我们就无法再访问子作用域上的 `$listeners` 了，由于这些作用域已经不再是作用域树上的一部分，它们自然也不会再接收到任何事件了。除非在应用代码中在某处泄漏了这些子作用域和监听器的引用，否则它们（所占用的内存）在稍后就会被当作垃圾进行回收。
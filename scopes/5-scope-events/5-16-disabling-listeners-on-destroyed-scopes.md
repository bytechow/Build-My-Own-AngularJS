### 禁用已销毁作用域上的 listener

#### Disabling Listeners On Destroyed Scopes

除了 `$destroy` 事件之外，销毁一个作用域的副作用还包括让事件监听器不会再被触发：

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

我们可以重新把 `$$listeners` 对象设置为一个空对象，这样就能有效地丢弃掉所有的事件监听器：

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

这样会让子作用域上的 `$listeners` 无法被访问，但由于这些作用域已经不在是作用域树的一部分，他们也不会再接收到任何事件了。除非在应用代码中存在引用泄漏的问题，否则这些子作用域和它们的监听器都会被当作垃圾进行回收。
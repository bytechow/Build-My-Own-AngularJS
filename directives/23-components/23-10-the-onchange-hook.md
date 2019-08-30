### 生命周期钩子函数——$onChanges（The $onChanges Hook）

最后要讲的生命周期钩子函数是`$onChanges`。这个钩子函数也许是所有钩子函数中最有用的，但也是目前为止开发起来最为复杂的一个钩子函数。

它的主要思路是在组件绑定数据发生改变时发出通知。也就是说，当组件中的一个或多个输入项发生改变时，应用开发者可以加入一些处理逻辑，如完成派生值的计算或其他需要在变化时处理的事情。

这个功能点也是之前并没有给出比较理想的解决方案的。在 Angular 1.5 版本出来之前，我们可以在指令控制器中添加`$watch`，然后在 watch 的 listener 函数中实现这一功能：

```js
$scope.$watch('ctrl.someInput', function(newValue) {
  ctrl.derivedValue = deriveValue(newValue);
});
```

这种 API 不仅不方便，而且会带来性能的损耗。我们对已经被指令编译器 watch 的东西再加一个 watcher，这真是没什么必要。对于`$onChanges`，它实际上是不必要的，因为现在框架已经可以告诉我们发生了变化。

`$onChanges`第一次调用是发生在组件被初始化后。它会提供组件所有属性和单向绑定数据的初始值。

_test/compile_spec.js_

```js
it('calls $onChanges with all bindings during init', function() {
  var changesSpy = jasmine.createSpy();
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      myBinding: '<',
      myAttr: '@'
    },
    controller: function() {
      this.$onChanges = changesSpy;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component my-binding="42" my-attr="43"></my-component>');
    $compile(el)($rootScope);
    expect(changesSpy).toHaveBeenCalled();
     var changes = changesSpy.calls.mostRecent().args[0];
    expect(changes.myBinding.currentValue).toBe(42);
    expect(changes.myBinding.isFirstChange()).toBe(true);
    expect(changes.myAttr.currentValue).toBe('43');
    expect(changes.myAttr.isFirstChange()).toBe(true);
  }); 
});
```

注意，传递给`$onChange`的变化值参数的结构是这样的：它会包含与绑定数据相匹配的 key。对每一个绑定数据来说，它都包含一个`currentValue`的 key，还有一个会返回`true`结果值的`isFirstValue`函数。
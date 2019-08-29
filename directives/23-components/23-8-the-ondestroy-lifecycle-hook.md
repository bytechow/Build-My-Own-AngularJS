### 生命周期钩子函数——$onDestroy（The $onDestroy Lifecycle Hook）

第二个要讲的钩子函数是`$onDestroy`。顾名思义，这个函数会在控制器将要被销毁时调用。跟`$onInit`相反，它会发生在指令作用域将要被销毁之时。

我们本来就可以对 scope 销毁事件进行监听，因为我们可以对 scope 的`$destroy`事件进行监听，这是我们在本书的第一部分已经实现的特性。

但之前实现的特性并没有提供一个良好的 API。而新的`$onDestroy`钩子函数会更加方便，因为我们不再需要考虑作用域了。

_test/compile\_spec.js_

```js
it('calls $onDestroy when the scope is destroyed', function() {
  var destroySpy = jasmine.createSpy();
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function() {
      this.$onDestroy = destroySpy;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    $rootScope.$destroy();
    expect(destroySpy).toHaveBeenCalled();
  });
});
```

在之前用于检查控制器是否含有`$onInit`的遍历中，我们也要加入对是否有`$onDestroy`钩子函数的检查。只要控制器注册了`$onDestory`钩子，我们就在对应的作用域中添加一个`$destroy`事件监听器。我们只是为应用开发者省却绑定事件这一步而已：

```js
_.forEach(controllers, function(controller) {
  // var controllerInstance = controller.instance;
  // if (controllerInstance.$onInit) {
  //   controllerInstance.$onInit();
  // }
  if (controllerInstance.$onDestroy) {
    (newIsolateScopeDirective ? isolateScope : scope).$on('$destroy', function() {
      controllerInstance.$onDestroy();
    });
  }
});
```




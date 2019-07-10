### (Accessing Required Controllers from The Directive Controller)

目前我们已经看到了注入的控制器能够通过指令链接函数的第四个参数被访问到。在 Angular 1.5 以前，这就是能够访问控制器实例的唯一方法了。但指令链接函数却并不是我们最需要访问控制器的情景。更需要它们的情景是指令控制器中，因为我们会在指令控制器中加入指令的业务逻辑。

从 Angular 1.5 版本开始，被注入的控制器也能作为指令控制器的一个属性，这是的我们可以更自然地访问它们。

```js
function MyController() {
  this.doSomething() {
    this.someRequiredController.doSomethingElse();
  }
}
```

这只会出现在使用对象形式注入时。更准确来说，只会出现在指令启用了`bindToController`属性的情况。下面是一个测试用例。

```js
it('attaches required controllers on controller when using object', function() {
  function MyController() {}
  var instantiatedController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: {
          myDirective: '^'
        },
        bindToController: true,
        controller: function() {
          instantiatedController = this;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-other-directive></div></div>');
    $compile(el)($rootScope);
    expect(instantiatedController.myDirective instanceof MyController).toBe(true);
  });
});
```

这些都会发生在节点链接函数初始化完所有指令控制器后。

```js
// _.forEach(controllers, function(controller) {
//   controller();
// });

_.forEach(controllerDirectives, function(controllerDirective, name) {

});
````

> 注意，这意味着当控制器构造函数在调用时，被注入的控制器还没准备好。

在循环里面，我们会检查一些基本条件，包括看`require`属性是否是一个对象，还要看指令是否启用了`bindToController`。

_src/compile.js_

```
```
### 在作用域上加入指令控制器（Attaching Directive Controllers on The Scope）

我们已经知道如何往控制器传递作用域对象了，其实我们也可以反过来为作用域添加一个控制器对象。这允许我们进行一个新的应用模式：通过`this`而不是`$scope`来公开控制器数据和函数，同时还可以让它们能够在 DOM 差值表达式使用，也能够在子指令和控制器中使用。

> [Todd Motto的一篇文章](http://toddmotto.com/digging-into-angulars-controller-as-syntax/)对这种应用模式有一个不错的介绍。之后我们再根据文章的内容对这种模式作进一步的解释，这里我们先关注搭建这种模式的基础。

当指令定义对象定义了一个`controllerAs`的 key 时，也就指定了添加到作用域的控制器对象的 key：

_test/compile\_spec.js_

```js
it('can be attached on the scope', function() {
  function MyController() {}
  var injector = createInjector(['ng',
    function($controllerProvider, $compileProvider) {
      $controllerProvider.register('MyController', MyController);
      $compileProvider.directive('myDirective', function() {
        return {
          controller: 'MyController',
          controllerAs: 'myCtrl'
        };
      });
    }
  ]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect($rootScope.myCtrl).toBeDefned();
    expect($rootScope.myCtrl instanceof MyController).toBe(true);
  });
});
```

在这个用例中，指令并没有要求获得一个继承的或是独立的作用域，所以获取这个控制器的作用域是`$rootScope`，这更方便我们进行测试。

要让这个测试用例通过，`$controller`函数需要再接收一个额外的可选参数，这个参数能定义作用域上的控制器的“标识符”。在节点连接函数中，如果我们调用`$controller`，我们也可以使用`controllerAs`的值：

_src/compile.js_

```js
_.forEach(controllerDirectives, function(directive) {
  // var locals = {
  //   $scope: scope,
  //   $element: $element,
  //   $attrs: attrs
  // };
  // var controllerName = directive.controller;
  // if (controllerName === '@') {
  //   controllerName = attrs[directive.name];
  // }
  $controller(controllerName, locals, directive.controllerAs);
});
```

> 这个参数实际上并不是给应用开发者直接使用的。它的存在只是为了支持指令编译器的`controllerAs`特性。
>
> 另外，这个实际上是`$controller`的第四个参数，而不是第三个。第三个参数会保留给另一个可选参数，在本章的后面会进行介绍。

如果提供了这个可选参数，`$controller`就会调用一个内部帮助函数来讲控制器实例添加到作用域上：

_src/controller.js_

```js
return function(ctrl, locals, identifer) {
  // if (_.isString(ctrl)) {
  //   if (controllers.hasOwnProperty(ctrl)) {
  //     ctrl = controllers[ctrl];
  //   } else if (globals) {
  //     ctrl = window[ctrl];
  //   }
  // }
  var instance = $injector.instantiate(ctrl, locals);
  if (identifer) {
    addToScope(locals, identifer, instance);
  }
  return instance;
};
```

这个`addToScope`函数会从给予的`locals`对象中找到作用域，并使用前面的标识符来把控制器实例添加到作用域上。如果提供了标志符但在`locals`对象中没有`$scope`将会抛出异常。

这个函数将会放到`controller.js`文件的最顶层：

```js
function addToScope(locals, identifer, instance) {
  if (locals && _.isObject(locals.$scope)) {
    locals.$scope[identifer] = instance;
  } else {
    throw 'Cannot export controller as ' + identifer +
      '! No $scope object provided via locals';
  }
}
```




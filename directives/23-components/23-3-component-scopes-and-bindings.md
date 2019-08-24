### 组件作用域与绑定数据（Component Scopes and Bindings）

像`restrict`一样，组件也帮应用开发者决定了作用域的定义：组件都会有独立作用域。这个事实毋庸置疑。

_test/compile_spec.js_

```js
it('has an isolate scope', function() {
  var componentScope;
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function($scope) {
      componentScope = $scope;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(componentScope).not.toBe($rootScope);
    expect(componentScope.$parent).toBe($rootScope);
    expect(Object.getPrototypeOf(componentScope)).not.toBe($rootScope);
  });
});
```

我们只需要在组件的指令工厂函数中定义即可：

_src/compile.js_

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    scope: {}
  }; 
}
```

组件也肯定可以有绑定数据。当前指令绑定数据的所有类型（属性类、单向绑定类、双向绑定类、表达式类），都适用于组件。只是我们并不是通过`scope`属性来定义绑定数据，而是通过一个叫`bindings`的属性。当然，这些绑定数据都会绑定到控制器上，而不是作用域上。

_test/compile_spec.js_

```js
it('may have bindings which are attached to controller', function() {
  var controllerInstance;
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      attr: '@',
      oneWay: '<',
      twoWay: '='
    },
    controller: function() {
      controllerInstance = this;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.b = 42;
    $rootScope.c = 43;
    var el = $('<my-component attr="a", one-way="b", two-way="c"></my-component>');
    $compile(el)($rootScope);
    
    expect(controllerInstance.attr).toEqual('a');
    expect(controllerInstance.oneWay).toEqual(42);
    expect(controllerInstance.twoWay).toEqual(43);
  });
});
```

要实现这个效果，我们可以利用控制器章节实现的一个便捷方法，也就是我们可以把一个对象作为`bindToController`的值来定义控制器绑定数据。我们直接将传入的`bindings`属性放到`bindToController`那里就可以了：

_src/compile.js_

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    scope: {},
    bindToController: options.bindings || {}
  }; 
}
```

另一样我们可以添加到组件作用域上的就是定义组件控制器的别名。它的工作方式跟常规指令是一样的，都是通过`controllerAs`属性：

_test/compile_spec.js_

```js
it('may use a controller alias with controllerAs', function() {
  var componentScope;
  var controllerInstance;
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function($scope) {
      componentScope = $scope;
      controllerInstance = this;
    },
    controllerAs: 'myComponentController'
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(componentScope.myComponentController).toBe(controllerInstance);
  });
});
````


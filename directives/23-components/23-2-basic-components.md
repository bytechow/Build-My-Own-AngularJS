### 基本组件（Basic Components）

那么如果组件就是指令，那他们究竟是哪一种指令呢？这是我们下面几页要介绍的内容。

我们将会在单元测试中新建很多组件，因此在我们开始之前，先弄一个帮助函数来帮助我们快速创建一个带有组件的注射器：

_test/compile_spec.js_

```js
function makeInjectorWithComponent(name, options) {
  return createInjector(['ng', function($compileProvider) {
    $compileProvider.component(name, options);
  }]);
}
```

关于组件，我们首先要知道它们是一个带有控制器的元素式组件。我们可以定义一个带一个控制器的组件，然后验证一下它在以元素方式出现时能不能被编译和链接。另外，我们还要检查一下这个控制是不是已经进行了实例化，并且`$element`参数可以注入到这个控制器中。

```js
it('are element directives with controllers', function() {
  var controllerInstantiated = false;
  var componentElement;
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function($element) {
      controllerInstantiated = true;
      componentElement = $element;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(controllerInstantiated).toBe(true);
    expect(el[0]).toBe(componentElement[0]);
  }); 
});
```

注意，与指令相比，组件有一个稍微更简化的注册 API。我们直接就给出组件定义对象。这也意味着没有组件工厂函数可以注入依赖，但它的设计理念就是，如果你需要注入依赖，你可以直接在控制器中注入。

另外，即使我们加入了`restrict`属性，组件也不能通过属性进行匹配。它真的就只能是元素。

_test/compile_spec.js_

```js
it('cannot be applied to an attribute', function() {
  var controllerInstantiated = false;
  var injector = makeInjectorWithComponent('myComponent', {
    restrict: 'A', // Will be ignored
    controller: function() {
      controllerInstantiated = true;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-component></div>');
    $compile(el)($rootScope);
    expect(controllerInstantiated).toBe(false);
  }); 
});
```

这些测试都表明了我们允许在组件选项对象中定义控制器，这个控制器会直接被传给指令工厂函数。另一方面，组件中不接受`restrict`属性，对应指令的`restrict`会硬编码为`E`。

_src/compile.js_

```js
this.component = function(name, options) {
  function factory() {
    return {
      restrict: 'E',
      controller: options.controller
    };
  }
  
  return this.directive(name, factory);
};
```
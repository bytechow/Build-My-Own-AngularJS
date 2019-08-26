### 组件模板（Component Templates）

像本章开头提到的，组件模式出现的关键原因就是把模板和控制器绑定到一起。所以，认为组件支持加入模板是十分合理的猜想。准确的猜想是：一个组件可以定义一个模板字符串。

_test/compile_spec.js_

```js
it('may have a template', function() {
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function() {
      this.message = 'Hello from component';
    },
    template: '{{ $ctrl.message }}'
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    expect(el.text()).toEqual('Hello from component');
  }); 
});
```

组件定义对象中的 template 属性可以直接传递给指令工厂函数。

```js
function factory() {
  return {
    // restrict: 'E',
    // controller: options.controller,
    // controllerAs: options.controllerAs ||
    //                 identifierForController(options.controller) ||
    //                 '$ctrl',
    // scope: {},
    // bindToController: options.bindings || {},
    template: options.template
  }; 
}
```

对模板 URL 也是一样的。组件可能会定义一个`templateUrl`，这会让组件通过 HTTP 获取一个模板：

_test/compile_spec.js_

```js
it('may have a templateUrl', function() {
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function() {
      this.message = 'Hello from component';
    },
    templateUrl: '/my_component.html'
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    requests[0].respond(200, {}, '{{ $ctrl.message }}');
    $rootScope.$apply();
    expect(el.text()).toEqual('Hello from component');
  });
});
```

要支持这个测试用例，我们需要在`describe('components')`测试块使用 Sinon 的模拟 HTTP 请求工具：

_test/compile_spec.js_

```js
describe('components', function() {

  var xhr, requests;
  beforeEach(function() {
    xhr = sinon.useFakeXMLHttpRequest();
    requests = [];
    xhr.onCreate = function(req) {
      requests.push(req);
    };
  });
  afterEach(function() {
    xhr.restore();
  });
  
  // ...
});
```

这种情况，我们也直接把`templateUrl`传递给指令工厂函数即可：

_src/compile.js_

```js
function factory() {
  return {
    // restrict: 'E',
    // controller: options.controller,
    // controllerAs: options.controllerAs ||
    //   identifierForController(options.controller) ||
    //   '$ctrl',
    // scope: {},
    // bindToController: options.bindings || {},
    // template: options.template,
    templateUrl: options.templateUrl
  };
}
```

当我们开发指令模板时，也介绍过另一种定义模板的方法，也就是使用一个函数作为`template`或`templateUrl`的值，而不直接使用模板字符串。这个函数会接收当前元素和`Attributes`对象作为参数，并返回一个模板字符串或模版 URL 字符串。在运行时，可以通过调用这个函数动态生成模板或模板字符串。

组件也支持这种方法，但有一点区别。调用组件中的`template`（或 templateUrl）函数时支持依赖注入，因此可以使用到各种依赖，一般指令就无法做到了。

_test/compile_spec.js_

```js
it('may have a template function with DI support', function() {
  var injector = createInjector(['ng', function($provide, $compileProvider) {
    $provide.constant('myConstant', 42);
    $compileProvider.component('myComponent', {
      template: function(myConstant) {
        return '' + myConstant;
      }
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(el.text()).toEqual('42');
  });
});
```

这个函数也可以包裹在一个数组中，也就是可以支持数组形式的依赖注入注解。

```js
it('may have a template function with array-wrapped DI', function() {
  var injector = createInjector(['ng', function($provide, $compileProvider) {
    $provide.constant('myConstant', 42);
    $compileProvider.component('myComponent', {
      template: ['myConstant', function(c) {
        return '' + c;
      }]
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(el.text()).toEqual('42');
  });
});
```
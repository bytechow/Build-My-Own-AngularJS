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

我们会把`template`属性值传递给一个新的帮助函数`makeinjectable`作为这个功能开发的第一步。在这个帮助函数中我们会做一血必要的预处理。

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
    template: makeInjectable(options.template),
    // templateUrl: options.templateUrl
  };
}
```

在这个函数里（我们会在`compile.js`文件的顶层作用域定义这个函数），我们会对模板进行检查，看它是不是函数或数组，如果是我们就对这个值进行处理：

_src/compile.js_

```js
function makeInjectable(template) {
  if (_.isFunction(template) || _.isArray(template)) {
  
  } else {
    return template;
  }
}
```

那我们可以做些什么呢？嗯，你可以回忆一下我们在依赖注入章节中介绍的`$injector`服务，里面有一个函数叫`invoke`，可以用来在调用任意函数时加入依赖注入的支持。这也就是我们现在需要的了。但首先我们需要先获取到`$injector`服务本身，因为在`component`方法中并没有提供这个服务。但因为我们是在定义一个指令工厂函数，这个函数支持依赖注入，我们可以直接在指令工厂函数中注入`$injector`，并把它传递给`makeInjectable`即可。

```js
function factory($injector) {
  return {
    // restrict: 'E',
    // controller: options.controller,
    // controllerAs: options.controllerAs ||
    //               identifierForController(options.controller) ||
    //               '$ctrl',
    // scope: {},
    // bindToController: options.bindings || {},
    template: makeInjectable(options.template, $injector),
    // templateUrl: options.templateUrl
  };
}
factory.$inject = ['$injector'];
```

然后，我们会动态创建一个指令模板函数。它会使用`$injector.invoke`对原始的组件模板函数进行调用：

```js
function makeInjectable(template, $injector) {
  if (_.isFunction(template) || _.isArray(template)) {
    return function() {
      return $injector.invoke(template, this);
    };
  } else {
    return template;
  }
}
```

但我们还缺漏了一些东西。一般指令模板函数都会传入指令元素和属性作为参数，因为它们可能包含让我们进行动态构建模板的信息。我们的组件模板函数目前还不能访问到这两个参数，因为它们是不能通过`$injector`服务进行注入的。如果我们试着去使用它们，结果会是报出依赖注入失败的信息：

_test/compile_spec.js_

```js
it('may inject $element and $attrs to template function', function() {
  var injector = createInjector(['ng', function($provide, $compileProvider) {
    $compileProvider.component('myComponent', {
      template: function($element, $attrs) {
        return $element.attr('copiedAttr', $attrs.myAttr);
      }
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component my-attr="42"></my-component>');
    $compile(el)($rootScope);
    expect(el.attr('copiedAttr')).toEqual('42');
  });
});
```

要修复这个问题也很简单，多亏了我们在`$injector.invoke`函数中对本地变量进行了支持。我们可以通过包裹指令模板函数的函数对这两个参数进行获取，然后把它们作为要注入到组件模板函数的本地函数，就实现了我们想要的效果了。

_src/compile.js_

```js
function makeInjectable(template, $injector) {
  if (_.isFunction(template) || _.isArray(template)) {
    return function(element, attrs) {
      return $injector.invoke(template, this, {
        $element: element,
        $attrs: attrs
      });
    };
  } else {
    return template;
  }
}
```

最后，由于对`templateUrl`与`template`的处理情况并无差别，我们可以使用相同方法来对`templateUrl`加入依赖注入的支持：

_test/compile_spec.js_

```js
it('may have a template function with DI support', function() {
  var injector = createInjector(['ng', function($provide, $compileProvider) {
    $provide.constant('myConstant', 42);
    $compileProvider.component('myComponent', {
      templateUrl: function(myConstant) {
        return '/template' + myConstant + ".html";
      }
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    expect(requests[0].url).toBe('/template42.html');
  });
});
```

我们可以同样把这个`templateUrl`属性值传递给`makeInjectable`函数，这样就能起效了。

_src/compile.js_

```js
function factory($injector) {
  return {
    // restrict: 'E',
    // controller: options.controller,
    // controllerAs: options.controllerAs ||
    //   identifierForController(options.controller) ||
    //   '$ctrl',
    // scope: {},
    // bindToController: options.bindings || {},
    // template: makeInjectable(options.template, $injector),
    templateUrl: makeInjectable(options.templateUrl, $injector)
  };
}
```
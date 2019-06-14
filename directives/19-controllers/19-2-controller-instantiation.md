### 实例化控制器（Controller Instantiation）

AngularJS 中的控制器通常是使用构造函数创建出来的。这种函数名称的第一个字母按照惯例是使用大写字母的，并会使用`new`运算符进行实例化：

```js
function MyController() {
  this.someField = 42;
}
```

与传统的实例化不同的是，应用开发者会提供构造函数并传递给框架中的`$controller`服务，`$controller`服务会在需要的时候对构造函数进行实例化调用。最简单的，就是给`$controller`服务传递一个构造函数作为函数，看调用返回的结果构造函数的实例。这就是我们第一个测试要做的事情，所以我们先新建一个测试文件：

_test/controller_spec.js_

```js
'use strict';

var publishExternalAPI = require('../src/angular_public');

var createInjector = require('../src/injector');

describe('$controller', function() {

  beforeEach(function() {
    delete window.angular;
    publishExternalAPI();
  });

  it('instantiates controller functions', function() {
    var injector = createInjector(['ng']);
    var $controller = injector.get('$controller');

    function MyController() {
      this.invoked = true;
    }

    var controller = $controller(MyController);
    
    expect(controller).toBeDefned();
    expect(controller instanceof MyController).toBe(true);
    expect(controller.invoked).toBe(true);
  });

});
```

这个测试会检查我们调用的返回值是不是我们传入的构造函数的实例，同时会确认生成的对象是否通过调用这个构造函数生成。

`$controller`最简单的实现方法自然是直接使用`new`运算符调用控制器构造函数。而考虑到需要依赖注入，`$controller`的工作会更为有趣。实际上，构造函数也是带有依赖地进行调用的：

_test/controlle_spec.js_

```js
it('injects dependencies to controller functions', function() {
  var injector = createInjector(['ng', function($provide) {
    $provide.constant('aDep', 42);
  }]);
  var $controller = injector.get('$controller');

  function MyController(aDep) {
    this.theDep = aDep;
  }

  var controller = $controller(MyController);
  
  expect(controller.theDep).toBe(42);
});
```

由这些测试，我们了解到`$controller`就是一个函数，因为我们可以直接对它进行调用。也就是说，控制器的 provider $get 方法的返回值会是一个函数：

_src/controller.js_

```js
// function $ControllerProvider() {

//   this.$get = function() {
  
    return function() {
      
    };
  
//   };

// }
```

这个函数可以接受一个构造函数作为参数，获取依赖注入，最终返回一个对应的实例。在`$injector`服务中，我们已经有了类似功能的 API——`instantiate`，我们可以直接使用：

_src/controller.js_

```js
this.$get = ['$injector', function($injector) {

  return function(ctrl) {
    return $injector.instantiate(ctrl);
  };
  
}];
```

并不是所有的控制器构造函数的参数都必须要提前在注射器中注册。我们可以在`$controller`的第二个参数中声明需要提前注册的依赖，使用一个对象包裹。这是我们经常会在应用的单元测试中使用的、用于提供一个作用域上下文的方式。

_test/controller_spec.js_

```js
it('allows injecting locals to controller functions', function() {
  var injector = createInjector(['ng']);
  var $controller = injector.get('$controller');

  function MyController(aDep) {
    this.theDep = aDep;
  }

  var controller = $controller(MyController, {
    aDep: 42
  });
  
  expect(controller.theDep).toBe(42);
});
```

幸运的是，`$injector.instantiate`也对此进行了支持：

_src/controller.js_

```js
// this.$get = ['$injector', function($injector) {

  return function(ctrl, locals) {
    return $injector.instantiate(ctrl, locals);
  };
  
// }];
```
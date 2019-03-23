### 注册指令（Registering Directives）

`$compile`服务的主要工作是将指令转换为 DOM。当然要转换必须先能拿到指令。也就是说，我们需要一种方法来注册指令。

跟 service、factory 和其他组件一样，指令的注册同样通过模块来完成。一个指令可以通过模块对象的`directive`方法进行注册。当注册好一个指令后，这个指令会自动加上`Directive`后缀，因此，如果我们注册了一个指令为`abc`，那注射器就会拥有一个名为`abcDirective`的后缀：

_src/compile_spec.js_

```js
'use strict';

var _ = require('lodash');
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');

describe('$compile', function() {

  beforeEach(function() {
    delete window.angular;
    publishExternalAPI();
  });

  it('allows creating directives', function() {
    var myModule = window.angular.module('myModule', []);
    myModule.directive('testing', function() {});
    var injector = createInjector(['ng', 'myModule']);
    expect(injector.has('testingDirective')).toBe(true);
  });

});
```

模块对象的`directive`方法实际上跟我们之前为过滤器创建的方法大同小异。`directive`方法也是会把通过`$compileProvider`的`directive`方法注册的任务进行排队：

_src/loader.js_

```js
var moduleInstance = {
//   name: name,
//   requires: requires,
//   constant: invokeLater('$provide', 'constant', 'unshift'),
//   provider: invokeLater('$provide', 'provider'),
//   factory: invokeLater('$provide', 'factory'),
//   value: invokeLater('$provide', 'value'),
//   service: invokeLater('$provide', 'service'),
//   decorator: invokeLater('$provide', 'decorator'),
//   flter: invokeLater('$flterProvider', 'register'),
  directive: invokeLater('$compileProvider', 'directive'),
  // confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
  // run: function(fn) {
  //   moduleInstance._runBlocks.push(fn);
  //   return moduleInstance;
  // },
  // _invokeQueue: invokeQueue,
  // _confgBlocks: confgBlocks
  // _runBlocks: []
};
```

目前来说，我们要做的就是在`$CompileProvider`的`directive`方法里面调用`$provider`注册一个对应该指令的 factory。为此，我们需要在`$CompileProvider`中注入`$provider`服务。同时，我们需要为`$CompileProvider`增加`$inject`属性，这样即使进行压缩也不会有问题：

_src/compile.js_

```js
function $CompileProvider($provide) {

  this.directive = function(name, directiveFactory) {
    $provide.factory(name + 'Directive', directiveFactory);
  };

  // this.$get = function() {

  // };

}
$CompileProvider.$inject = ['$provide'];
```

可以看到，当我们注册指令时，实际上会在注射器中加入一个 factory 服务。指令 factory 还有一个其他 factory 不具备的特点：可以有多个指令共用同一个名称。

_test/compile\_spec.js_

```js
it('allows creating many directives with the same name', function() {
  var myModule = window.angular.module('myModule', []);
  myModule.directive('testing', _.constant({ d: 'one' }));
  myModule.directive('testing', _.constant({ d: 'two' }));
  var injector = createInjector(['ng', 'myModule']);

  var result = injector.get('testingDirective');
  expect(result.length).toBe(2);
  expect(result[0].d).toEqual('one');
  expect(result[1].d).toEqual('two');
});
```

在这个测试中，我们会注册两个指令，这两个指令的名称都是`testing`，然后再看注射器的 testingDirective 会返回什么结果。我们希望这个结果会是一个包含两个指令的数组。

> 目前指令本身不会做什么，就只是对象字面量。

所以，不像其他组件，我们无法通过声明一个同名指令来重写指令。要改变一个已经存在的指令，你需要使用装饰器（decorator）。

允许多个指令共用名称是为了能让这些指令能够匹配对应的 DOM 元素和属性。如果 Angular 限定指令名称必须是唯一的，就无法让两个指令都匹配到同一个元素。这跟 jQuery 不允许同一选择器获取不同结果类似，这确实需要严格限制。

我们要做的是在`$CompileProvider.directive`中新增一个内部变量`hasDirectives`以记录指令的注册关系，让每个指令名称对应一个可以存储多个指令 factory 的数组：

_src/compile.js_

```js
// function $CompileProvider($provide) {

  var hasDirectives = {};

  // this.directive = function(name, directiveFactory) {
    if (!hasDirectives.hasOwnProperty(name)) {
      hasDirectives[name] = [];
    }
    hasDirectives[name].push(directiveFactory);
  // };

//   this.$get = function() {

//   };
// }
```

我们调用`$provider`注册的是一个函数，这个函数将会在内部变量`hasDirectives`中查找要加载的指令 factory，然后用`$injector.invoke`调用即可，最终用数组存放调用结果，最终返回这个结果：

```js
// this.directive = function(name, directiveFactory) {
//   if (!hasDirectives.hasOwnProperty(name)) {
//     hasDirectives[name] = [];
    $provide.factory(name + 'Directive', ['$injector', function($injector) {
      var factories = hasDirectives[name];
      return _.map(factories, $injector.invoke);
    }]);
//   }
//   hasDirectives[name].push(directiveFactory);
// };es[name].push(directiveFactory);
};
```

我们现在需要引入 LoDash 了：

```js
'use strict';

var _ = require('lodash');
```

对应用开发者来说，他们基本不需要在应用代码中注入指令，因为一般我们都会通过 DOM 编译的方式来使用指令。如果确实需要通过依赖注入获取一个指令，也是可以的。当然，由于指令 factory 的特性，我们获取到的是一个数组。

还有一种特殊情况需要我们处理。由于我们使用了`hasOwnProperty`，就如上面章节提及的，我们需要防止用户注册名为`hasOwnProperty`的指令，而意外重写了 `hasDirectives`的`hasOwnProperty`方法，导致程序无法正常运行：

_test/compile\_spec.js_

```js
it('does not allow a directive called hasOwnProperty', function() {
  var myModule = window.angular.module('myModule', []);
  myModule.directive('hasOwnProperty', function() {});
  expect(function() {
    createInjector(['ng', 'myModule']);
  }).toThrow();
});
```

我们直接通过字符串的匹配来进行规避：

_src/compile.js_

```js
// this.directive = function(name, directiveFactory) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid directive name';
  }
//   if (!hasDirectives.hasOwnProperty(name)) {
//     hasDirectives[name] = [];
//     $provide.factory(name + 'Directive', ['$injector', function($injector) {
//       var factories = hasDirectives[name];
//       return _.map(factories, $injector.invoke);
//     }]);
//   }
//   hasDirectives[name].push(directiveFactory);
// };
```

我们还得考虑指令注册的另一个特性：我们可以通过快捷方式同时注册多个指令。我们可以通过往`directive`方法传递一个对象作为参数实现。属性名会被指定为指令名称，属性值就是对应的指令 factory：

_test/compile\_spec.js_

```js
it('allows creating directives with object notation', function() {
  var myModule = window.angular.module('myModule', []);
  myModule.directive({
    a: function() {},
    b: function() {},
    c: function() {}
  });
  var injector = createInjector(['ng', 'myModule']);

  expect(injector.has('aDirective')).toBe(true);
  expect(injector.has('bDirective')).toBe(true);
  expect(injector.has('cDirective')).toBe(true);
});
```

在`directive`方法的实现代码中，我们需要通过判断传入的第一个参数类型进行区分处理：

_src/complie.js_

```js
this.directive = function(name, directiveFactory) {
  if (_.isString(name)) {
    // if (name === 'hasOwnProperty') {
    //   throw 'hasOwnProperty is not a valid directive name';
    // }
    // if (!hasDirectives.hasOwnProperty(name)) {
    //   hasDirectives[name] = [];
    //   $provide.factory(name + 'Directive', ['$injector', function($injector) {
    //     var factories = hasDirectives[name];
    //     return _.map(factories, $injector.invoke);
    //   }]);
    // }
    // hasDirectives[name].push(directiveFactory);
  } else {

  }
};
```

如果检测到参数是对象类型，就要对对象进行遍历，然后递归调用 directive 方法进行注册：

```js
// this.directive = function(name, directiveFactory) {
//   if (_.isString(name)) {
//     if (name === 'hasOwnProperty') {
//       throw 'hasOwnProperty is not a valid directive name';
//     }
//     if (!hasDirectives.hasOwnProperty(name)) {
//       hasDirectives[name] = [];
//       $provide.factory(name + 'Directive', ['$injector', function($injector) {
//         var factories = hasDirectives[name];
//         return _.map(factories, $injector.invoke);
//       }]);
//     }
//     hasDirectives[name].push(directiveFactory);
  // } else {
    _.forEach(name, _.bind(function(directiveFactory, name) {
      this.directive(name, directiveFactory);
    }, this));
//   }
// };
```




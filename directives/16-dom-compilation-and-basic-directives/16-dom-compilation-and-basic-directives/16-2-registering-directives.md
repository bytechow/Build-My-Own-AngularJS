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

```
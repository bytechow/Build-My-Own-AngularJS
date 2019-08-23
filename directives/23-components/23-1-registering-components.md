### 注册组件（Registering Components）

新增特性，我们还是安装之前的管理：先对模块 API 进行扩展，以便实现功能的注册。我们需要的模块方法就叫做`component`，它会接收组件名称和组件对象两个参数，同时按照传入的组件名称生成一个指令。

我们先在指令编译器的测试文件中新增对组件功能的测试集：

_test/compile_spec.js_

```js
describe('components', function() {

  it('can be registered and become directives', function() {
    var myModule = window.angular.module('myModule', []);
    myModule.component('myComponent', {});
    var injector = createInjector(['ng', 'myModule']);
    expect(injector.has('myComponentDirective')).toBe(true);
  });

});
```

注册过程是从模块加载器开始的。模块实例的`component`方法会对`$compileProvider`中的`component`方法的调用进行存储与排队。

_src/loader.js_

```js
var moduleInstance = {
  // name: name,
  // requires: requires,
  // constant: invokeLater('$provide', 'constant', 'unshift'),
  // provider: invokeLater('$provide', 'provider'),
  // factory: invokeLater('$provide', 'factory'),
  // value: invokeLater('$provide', 'value'),
  // service: invokeLater('$provide', 'service'),
  // decorator: invokeLater('$provide', 'decorator'),
  // filter: invokeLater('$filterProvider', 'register'),
  // directive: invokeLater('$compileProvider', 'directive'),
  // controller: invokeLater('$controllerProvider', 'register'),
  component: invokeLater('$compileProvider', 'component'),
  
  // config: invokeLater('$injector', 'invoke', 'push', configBlocks),
  // run: function(fn) {
  //   moduleInstance._runBlocks.push(fn);
  //   return moduleInstance;
  // },
  // _invokeQueue: invokeQueue,
  // _configBlocks: configBlocks,
  // _runBlocks: []
};
```

这个方法会在`$CompileProvider`中引入。你可以把它放置在`this.directive`和`this.$get`两段代码之间：

_src/compile.js_

```js
this.component = function(name, options) {

};
```

我们希望这个方法的调用，能够生成一个对应的指令。我们只需要在方法里面直接调用`directive`方法就可以了。我们会向`directive`方法传入动态生成的指令工厂函数：

```js
this.component = function(name, options) {
  function factory() {
    return {

    }; 
  }

  return this.directive(name, factory);
};
```

作为构建开始的第一步，这样就可以了。组件真的就是一个指令而已！
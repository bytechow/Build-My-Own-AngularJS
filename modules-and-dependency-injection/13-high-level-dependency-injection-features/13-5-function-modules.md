### 函数形式的模块（Function Modules）

正如我们所看到的，模块就是一个可以注册应用组件的对象。它内部管理着一个会在模块加载时执行的任务队列。

其实还有一种注册模块的方式：一个模块可以用函数的形式进行注册，当这个模块被载入时会从 provider injector 注入。

现在我们定义 myModule 为一个普通的模块对象。它只拥有一个依赖，它依赖的就是函数形式的模块：

test/injector\_spec.js

```js
it('runs a function module dependency as a confg block', function() {
    var functionModule = function($provide) {
        $provide.constant('a', 42);
    };
    window.angular.module('myModule', [functionModule]);
    var injector = createInjector(['myModule']);
    expect(injector.get('a')).toBe(42);
});
```

当然，我们也允许使用行内式注入：

test/injector\_spec.js

```js
it('runs a function module with array injection as a confg block', function() {
    var functionModule = ['$provide', function($provide) {
        $provide.constant('a', 42);
    }];
    window.angular.module('myModule', [functionModule]);
    var injector = createInjector(['myModule']);
    expect(injector.get('a')).toBe(42);
});
```

函数形式的模块手机上就是 config 服务，一个会被注入 provider 的服务。它们的不同仅在于定义位置：config 服务会被注册到一个模块中，而函数式模块是模块的依赖。

现在如果我们需要加载模块，我们不再假设模块就是一个字符串了，它又可能是一个函数或者数组，我们可以都通过 providerInjector.invoke 来进行调用：

src/injector.js

```js
.forEach(modulesToLoad, function loadModule(module) {
    if (_.isString(module)) {
        if (!loadedModules.hasOwnProperty(module)) {
            loadedModules[module] = true;
            module = window.angular.module(module);
            _.forEach(module.requires, loadModule);
            runInvokeQueue(module._invokeQueue);
            runInvokeQueue(module._confgBlocks);
            runBlocks = runBlocks.concat(module._runBlocks);
        }
    } else if (_.isFunction(module) || _.isArray(module)) {
        providerInjector.invoke(module);
    }
});
```




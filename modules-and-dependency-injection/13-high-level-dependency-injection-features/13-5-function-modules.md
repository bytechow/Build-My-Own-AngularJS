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

当你有一个函数式模块时，你也可以在里面返回一个函数，这个函数会作为 run 服务执行。这个小细节允许定义“点对点”的模块和相关的 run 服务，这在单元测试里面很有用。

test/injector\_spec.js

```js
it('supports returning a run block from a function module', function() {
  var result;
  var functionModule = function($provide) {
    $provide.constant('a', 42);
    return function(a) {
      result = a;
    };
  };
  window.angular.module('myModule', [functionModule]);
  createInjector(['myModule']);
  expect(result).toBe(42);
});
```

当函数式模块被执行时，我们需要需要把函数返回值放到 run 服务任务队列一样。当然，是否有返回值目前是可选的，也就意味着我们需要应对返回值为 undefined 的情况。我们可以利用 Lodash 中的 \_.compact 方法筛选掉值为 undefined 的数组元素：

src/injector.js

```js
var runBlocks = [];
_.forEach(modulesToLoad, function loadModule(module) {
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
        runBlocks.push(providerInjector.invoke(module));
    }
});
_.forEach(_.compact(runBlocks), function(runBlock) {
    instanceInjector.invoke(runBlock);
});
```

要加载函数式模块，我们还有一些工作要做。在第9章中，我们提到了模块的单例化——即使被引入多次，也只会加载一次。为此，我们使用了一个名为 loadedModules 的对象来记录已经加载过的模块。

但对于函数式模块，我们就无法使用这个 loadedModules 对象检测是否重复加载。也就是说，如果我们引入函数式依赖两次，它就真的会被实例化两次：

 test/injector_spec.js

```js
it('only loads function modules once', function() {
    var loadedTimes = 0;
    var functionModule = function() {
        loadedTimes++;
    };
    window.angular.module('myModule', [functionModule, functionModule]);
    createInjector(['myModule']);
    expect(loadedTimes).toBe(1);
});
```

我们没办法将函数作为对象的属性名，所以目前的函数式模块无法记录在 loadedModules 里。这里，我们就需要用到一个新的数据结构——HashMap。

在 ES2015（ES6）发布以前，JavaScript 还没有支持这种数据结构，所以 Angular 框架实现了 HashMap。我们将会在 loadedModules 应用这个数据结构。尽管还有很多的 JavaScript 第三方库支持 HashMap，我们还是会采用 Angular 本身的实现方案，毕竟对于我们了解原理已经足够了。接下来，我们会先暂时放下依赖注入的讲解，尝试一下自己实现 HashMap。


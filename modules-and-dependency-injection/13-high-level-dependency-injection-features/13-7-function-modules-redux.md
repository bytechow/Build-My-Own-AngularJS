### 函数式模块状态管理器（Function Modules Redux）

有了 HashMap，我们终于可以解决 injector_spec.js 中的还没解决的单元测试了，也就是我们可以保证函数式模块也只会被加载一次。

首先我们要引入 HashMap：

```js
'use strict';

var _ = require('lodash');
var HashMap = require('./hash_map').HashMap;
```

接着，我们会将 loadedModules 变量变成一个 HashMap 实例：

```js
var loadedModules = new HashMap();
```

最后，我们需要在 createInjector 遍历加载模块时，使用 HashMap 的 get 和 put 方法：

```js
_.forEach(modulesToLoad, function loadModule(module) {
  if (!loadedModules.get(module)) {
    loadedModules.put(module, true);
    if (_.isString(module)) {
      module = window.angular.module(module);
      _.forEach(module.requires, loadModule);
      runInvokeQueue(module._invokeQueue);
      runInvokeQueue(module._confgBlocks);
      runBlocks = runBlocks.concat(module._runBlocks);
    } else if (_.isFunction(module) || _.isArray(module)) {
      runBlocks.push(providerInjector.invoke(module));
    }
  }
});
```
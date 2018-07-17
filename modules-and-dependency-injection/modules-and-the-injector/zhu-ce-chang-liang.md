### 注册常量（Registering A Constant）

我们要接触的第一个 Angular 应用组件是常量（constants）。通过常量组件，我们可以在模块中注册一些简单值，比如数字、字符串、对象、函数等。

当我们在模块中注册了一个常量，并以该模块作为参数生成注射器，我们就可以使用 injector 的 has 方法来检查常量是否成功注册到该模块中：

test/injector\_spec.js

```js
it('has a constant that has been registered to a module', function() {
  var module = window.angular.module('myModule', []);
  module.constant('aConstant', 42);
  var injector = createInjector(['myModule']);
  expect(injector.has('aConstant')).toBe(true);
});
```

现在我们终于看到了从定义模块到创建该模块注射器的一个完整过程。我们可以看到一个有趣的现象，我们传递给 createInjector 的是模块的名称，而不是模块实例的引用，而最终我们会在 angular.module 中查找对应的模块实例

为了让检测更严谨，我们希望当调用 has 方法无法找到模块时会返回 false：

test/injector\_spec.js

```js
it('does not have a non-registered constant', function() {
  var module = window.angular.module('myModule', []);
  var injector = createInjector(['myModule']);
  expect(injector.has('aConstant')).toBe(false);
});
```

那么问题来了，我们应该怎么做才能让注册在模块中的常量能在注射器中使用？首先，我们需要一个在模块实例中新增一个注册方法：

src/loader.js

```js
var createModule = function(name, requires, modules) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var moduleInstance = {
    name: name,
    requires: requires,
    constant: function(key, value) {}
  };
  modules[name] = moduleInstance;
  return moduleInstance;
};
```

我们需要知道一条通用的规则——模块实际上不持有任何应用组件。他们只是保存创建应用组件的"食谱"（recipes），应用组件实际上被保存在注射器中。


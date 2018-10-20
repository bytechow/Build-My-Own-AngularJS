### 严格模式（Strict Mode）

推断式依赖注入是 Angular 中最有争议的特性之一，除了因为它本身是一种“黑魔法”（应用者难以想象到这是如何实现的）以外，也因为使用这种依赖注入的方式可能带来一些实际问题。比如在发布应用之前，我们很可能需要诸如 UglifyJS 或 Closure Compiler 之类的压缩工具对 JavaScript 源码进行压缩，而压缩就意味着会对源代码进行修改。对于推断式注解来说，它必须要依靠函数声明本身的参数列表来获取对应的依赖，但参数列表很可能会在压缩过程中发生改变，这就会使得依赖注入无法正常进行。

因此，很多开发者选择不使用这种无须注解的依赖注入方式（或者使用 ng-annotate 服务提前生成注解）。Angular 也提供了一种检测方案，能够限制在代码中使用推断式注入的行为，一旦在运行过程中发现有这种行为，会抛出一个错误进行提醒。

我们可以通过在调用 createInjector 函数时，传入一个布尔值来控制是否开启严格模式：

test/injector\_spec.js

```js
it('throws when using a non-annotated fn in strict mode', function() {
  var injector = createInjector([], true);
  var fn = function(a, b, c) { };
  expect(function() {
    injector.annotate(fn);
  }).toThrow();
});
```

同时，我们希望传入的第二个参数必须是 true，而不是其他真值（truthy）:

```js
function createInjector(modulesToLoad, strictDi) {
  var cache = {};
  var loadedModules = {};
  strictDi = (strictDi === true);
  // ...
}
```

在 annotate 方法中，我们在提取函数参数之前，检查当前注射器是否采用严格模式：

src/injector.js

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else if (fn.$inject) {
    return fn.$inject;
  } else if (!fn.length) {
    return [];
  } else {
    if (strictDi) {
      throw 'fn is not using explicit annotation and ' +
        'cannot be invoked in strict mode';
    }
    var source = fn.toString().replace(STRIP_COMMENTS, '');
    var argDeclaration = source.match(FN_ARGS);
    return _.map(argDeclaration[1].split(','), function(argName) {
      return argName.match(FN_ARG)[2];
    });
  }
}
```

如果你有压缩代码的需要，开启严格模式是绝对有必要的。这样即使你在代码中不小心使用了推断式注入，你也能从抛出的错误得到一个明确的提示，而不是告诉你要查找的某个依赖不存在，而这个依赖你根本没有注册过。


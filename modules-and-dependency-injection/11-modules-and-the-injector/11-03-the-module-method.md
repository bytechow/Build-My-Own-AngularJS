### module 方法（The module Method）

我们要介绍的第一个在 angular 全局对象的方法是 module，这个方法将会在本章以及之后的章节被大量使用。首先，我们断言这个方法是存在的：

test/loader\_spec.js

```js
it('exposes the angular module function', function() {
  setupModuleLoader(window);
  expect(window.angular.module).toBeDefined();
});
```

就像全局对象 angular 一样，module 方法也应该是单例的：

```js
it('exposes the angular module function just once', function() {
  setupModuleLoader(window);
  var module = window.angular.module;
  setupModuleLoader(window);
  expect(window.angular.module).toBe(module);
})
```

现在我们可以重用 ensure 函数：

```js
function setupModuleLoader(window) {
  var ensure = function(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  };
  var angular = ensure(window, 'angular', Object);
  ensure(angular, 'module', function() {
    return function() {};
  });
}
```




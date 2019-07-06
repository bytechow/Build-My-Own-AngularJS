### 自注入的指令（Self-Requiring Directives）

当一个指令定义了自己的控制器，但并没有引入其他指令控制器，它默认会接收自己的控制器对象作为链接函数的第四个参数。就像是指令自己引用自己一样，是一个便利的小特性：

_test/compile_spec.js_

```js
it('requires itself if there is no explicit require', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController,
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```

指令“注入自身”就如其字面的意思一样。当指令定义已经注册好，该指令只有`controller`属性而没有`require`属性，那`require`属性值就被默认为该指令本身的名字。这样，我们就会去查找自身的控制器，最终让测试用例通过：

_src/compile.js_

```js
function getDirectiveRequire(directive, name) {
  var require = directive.require || (directive.controller && name);
  // if (!_.isArray(require) && _.isObject(require)) {
  //   _.forEach(require, function(value, key) {
  //     if (!value.length) {
  //       require[key] = key;
  //     }
  //   });
  // }
  // return require;
}
```

接下来，我们只需要在调用`getDirectiveRequire`时传入指令名称即可：

```js
directive.require = getDirectiveRequire(directive, name);
```
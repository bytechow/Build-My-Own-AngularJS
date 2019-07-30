### 在控制器里的 transclusion（Transclusion in Controllers）

我们也可以在指令控制器里调用 transclusion 函数，是在链接函数里面调用 transclusion 函数的替代方式。我们可以通过注入`$transclude`参数来获取 transclusion 函数：

_test/compile_spec.js_

```js
it('makes contents available to controller', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        template: '<div in-template></div>',
        controller: function($element, $transclude) {
          $element.fnd('[in-template]').append($transclude());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');
    $compile(el)($rootScope);
    
    expect(el.fnd('> [in-template] > [in-transclude]').length).toBe(1);
  });
});
```

在构建控制器之前，我们可以把绑定了 scope 的 transclusion 函数添加到控制器的本地对象中去：

_src/compile.js_

```js
if (controllerDirectives) {
  _.forEach(controllerDirectives, function(directive, directiveName) {
    var locals = {
      // $scope: directive === newIsolateScopeDirective ? isolateScope : scope,
      // $element: $element,
      $transclude: scopeBoundTranscludeFn,
      // $attrs: attrs
    };
    // var controllerName = directive.controller;
    // if (controllerName === '@') {
    //   controllerName = attrs[directive.name];
    // }
    // var controller =
    //   $controller(controllerName, locals, true, directive.controllerAs);
    // controllers[directive.name] = controller;
    // $element.data('$' + directive.name + 'Controller', controller.instance);
  });
}
```

这意味着指令链接函数的第五个参数，还有指令控制器中的`$transclude`参数会提供同一样事物：绑定了 scope 的 transclusion 函数。
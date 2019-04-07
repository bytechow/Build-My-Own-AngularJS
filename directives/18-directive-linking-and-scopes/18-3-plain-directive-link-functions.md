### 普通的指令链接函数（Plain Directive Link Functions）

其实我们在开发过程中常常不会在`compile`加入处理，而会在`link`函数中放所有的处理逻辑。而 Angular 在指令的定义对象中提供了一个便利的接口，我们可以使用一个`link`属性来直接定义指令的链接函数，跳过`compile`那一步：

_test/compile_spec.js_

```js
it('supports link function in directive defnition object', function() {
  var givenScope, givenElement, givenAttrs;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      link: function(scope, element, attrs) {
        givenScope = scope;
        givenElement = element;
        givenAttrs = attrs;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(givenScope).toBe($rootScope);
    expect(givenElement[0]).toBe(el[0]);
    expect(givenAttrs).toBeDefned();
    expect(givenAttrs.myDirective).toBeDefned();
  });
});
```

我们在注册指令工厂加入这个“语法糖”。如果一个指令定义对象没有指定`compile`属性，但用`link`属性直接指定了链接函数，我们就把指令的编译函数替换为一个直接返回该链接函数的函数。而指令编译器对这种情况是一视同仁的：

_src/compile.js_

```js
$provide.factory(name + 'Directive', ['$injector', function($injector) {
  // var factories = hasDirectives[name];
  // return _.map(factories, function(factory, i) {
  //   var directive = $injector.invoke(factory);
  //   directive.restrict = directive.restrict || 'EA';
  //   directive.priority = directive.priority || 0;
    if (directive.link && !directive.compile) {
      directive.compile = _.constant(directive.link);
    }
//     directive.name = directive.name || name;
//     directive.index = i;
//     return directive;
//   });
// }]);
```
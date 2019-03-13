### 对元素式指令使用前缀（Using Prefixes with Element Directives）

我们已经看到指令会应用到同名元素的情况。本章下面几节内容，我们会看到另外几种匹配的类型。

首先，Angular 允许我们在匹配的元素名称前面使用`x`和`data`前缀：

```html
<x-my-directive></x-my-directive>
<data-my-directive></data-my-directive>
```

另外，除了连字符以外，我们还可以使用冒号或下划线作为前缀与指令名之间的分隔符：

```html
<x:my-directive></x:my-directive>
<x_my-directive></x_my-directive>
```

综上，我们就有六种为元素添加前缀的方式。要将这六种情况都进行单元测试，我们需要进行遍历，并对每一种情况生成一个测试的 block：

_test/compile_spec.js_

```js
_.forEach(['x', 'data'], function(prefx) {
  _.forEach([':', '-', '_'], function(delim) {

    it('compiles element directives with ' + prefx + delim + ' prefx', function() {
      var injector = makeInjectorWithDirectives('myDir', function() {
        return {
          compile: function(element) {
            element.data('hasCompiled', true);
          }
        };
      });
      injector.invoke(function($compile) {
        var el = $('<' + prefx + delim + 'my-dir></' + prefx + delim + 'my-dir>');
        $compile(el);
        expect(el.data('hasCompiled')).toBe(true);
      });
    });
    
  });
});
```

我们将会在`compile.js`文件中引入一个顶层函数来处理带前缀匹配的情况。它会接收元素名称作为参数值，然后返回一个经过“标准化”后的指令名称。这个过程包括去除前缀和转换为驼峰形式：

_src/compile.js_

```js
function directiveNormalize(name) {
  return _.camelCase(name.replace(PREFIX_REGEXP, ''));
}
```

下面这个正则表达式可以同时匹配`x`和`data`及它们与三种分隔符产生的各种组合：

```js
var PREFIX_REGEXP = /(x[\:\-_]|data[\:\-_])/i;
```

在`collectDirectives`中，我们现在会把之前的`_.camelCase`替换为`directiveNormalize`：

```js
function collectDirectives(node) {
  var directives = [];
  var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
  addDirective(directives, normalizedNodeName);
  return directives;
}
```
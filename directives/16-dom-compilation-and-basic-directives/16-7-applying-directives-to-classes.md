### 编译CSS样式类式指令（Applying Directives to Classes）

第三种应用指令的方法是通过 CSS 的样式类进行匹配。我们只需要使用一个匹配指令的样式名称就可以应用该指令了：

_test/compile_spec.js_

```js
it('compiles class directives', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div class="my-directive"></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
});
```

样式类式指令与属性式指令一样，都可以对同一个元素使用多个指令：

```js
it('compiles several class directives in an element', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        compile: function(element) {
          element.data('hasCompiled', true);
        }
      };
    },
    mySecondDirective: function() {
      return {
        compile: function(element) {
          element.data('secondCompiled', true);
        }
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div class="my-directive my-second-directive"></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
    expect(el.data('secondCompiled')).toBe(true);
  });
});
```

样式类式的指令还可以有跟之前的元素式、属性式指令相同的前缀（但是不会用到`ng-attr`前缀）：

```js
it('compiles class directives with prefxes', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div class="x-my-directive"></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
});
```

要实现基于样式类的匹配，我们需要遍历每个节点的`classList`属性，并对标准化后的样式名称进行检查，看是否有与指令匹配：

```js
function collectDirectives(node) {
//   var directives = [];
//   var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
//   addDirective(directives, normalizedNodeName);
//   _.forEach(node.attributes, function(attr) {
//     var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
//     if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
//       normalizedAttrName =
//         normalizedAttrName[6].toLowerCase() +
//         normalizedAttrName.substring(7);
//     }
//     addDirective(directives, normalizedAttrName);
//   });
  _.forEach(node.classList, function(cls) {
    var normalizedClassName = directiveNormalize(cls);
    addDirective(directives, normalizedClassName);
  });
//   return directives;
}
```

> AngularJS 实际上不会使用`classList`，因为它是一个 HTML5 的特性，还没被老浏览器支持。AngularJS 的替代做法是对元素的`className`进行token拆分（以获得与 classList 类似的效果）。
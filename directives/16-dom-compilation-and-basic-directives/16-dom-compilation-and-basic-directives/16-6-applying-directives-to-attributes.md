### 编译属性式指令（Applying Directives to Attributes）

匹配指令与 DOM 不止匹配元素名这一种方式。下面我们会介绍第二种方法，那就是属性名匹配。这可能算是 Angular 应用中使用指令最便利的方式：

_test/compile\_spec.js_

```js
it('compiles attribute directives', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
});
```

元素式指令使用前缀的规则同样适用于属性式指令。例如，`x:`前缀就是被允许的：

```js
it('compiles attribute directives with prefxes', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div x:my-directive></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
});
```

另外，对一个元素应用多个属性指令也是很正常的：

```js
it('compiles several attribute directives in an element', function() {
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
    var el = $('<div my-directive my-second-directive></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
    expect(el.data('secondCompiled')).toBe(true);
  });
});
```

我们还可以对同一个元素同时使用元素式和属性式指令：

```js
it('compiles both element and attributes directives in an element', function() {
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
    var el = $('<my-directive my-second-directive></my-directive>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
    expect(el.data('secondCompiled')).toBe(true);
  });
});
```

收集属性式指令的函数同样是`collectDirectives`，之前我们就是在这按照元素名匹配指令。在这里，我们会遍历当前节点的所有属性，并找出是否有指令与它们相匹配，而在查找之前我们需要对属性名称进行标准化：

_src/compile.js_

```js
function collectDirectives(node) {
  var directives = [];
  var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
  addDirective(directives, normalizedNodeName);
  _.forEach(node.attributes, function(attr) {
    var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
    addDirective(directives, normalizedAttrName);
  });
  return directives;
}
```

Angular 的属性式指令还支持一种特殊的前缀`ng-attr`：

_test/compile\_spec.js_

```js
it('compiles attribute directives with ng-attr prefx', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div ng-attr-my-directive></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
})
```

`ng-attr`前缀还能添加到已经包含了一个上面提及的六种前缀之一的属性上：

```js
it('compiles attribute directives with data:ng-attr prefx', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div data:ng-attr-my-directive></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
});
```

对于这种情况，我们会对标准化后的指令名称进行检测，看它开头的格式是否符合一个`ngAttr`后面跟着一个大写字母。如果是，我们就去掉`ngAttr`这六个字母，并把紧跟着的这个大写字母再变成小写：

_src/compile.js_

```js
function collectDirectives(node) {
  var directives = [];
  var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
  addDirective(directives, normalizedNodeName);
  _.forEach(node.attributes, function(attr) {
    var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
    if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
      normalizedAttrName =
        normalizedAttrName[6].toLowerCase() +
        normalizedAttrName.substring(7);
    }
    addDirective(directives, normalizedAttrName);
  });
  return directives;
}
```




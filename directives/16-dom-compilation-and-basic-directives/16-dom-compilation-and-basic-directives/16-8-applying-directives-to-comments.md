### 编译注释式指令（Applying Directives to Comments）

最后也是最令人难懂的一种指令匹配方式就是对 HTML 中的注释进行匹配。我们可以通过`directive:`前缀和指令名称来匹配指令：

```html
<!-- directive: my-directive -->
```

我们按照这种方式编写测试用例：

_test/compile_spec.js_

```js
it('compiles comment directives', function() {
  var hasCompiled;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        hasCompiled = true;
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<!-- directive: my-directive -->');
    $compile(el);
    expect(hasCompiled).toBe(true);
  });
});
```

> 这里我们会使用一个变量`hasCompiled`来进行标记，而不采用之前在 DOM 节点上加入 data 属性的方式，因为 jQuery 并不支持在注释节点上加入数据。

目前，我们在`collectDirectives`函数中传入的是一个 DOM 节点。这个 DOM 节点可能是一个元素节点，也可能是一个注释节点。我们需要将此处的处理逻辑改变成根据节点的类型来区分对待，也就是根据节点的`nodeType`属性。

我们将之前对元素节点的处理逻辑都放到一个 if 分支，而另一个分支用来处理注释节点：

_src/compile.js_

```js
function collectDirectives(node) {
  // var directives = [];
  if (node.nodeType === Node.ELEMENT_NODE) {
    // var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
    // addDirective(directives, normalizedNodeName);
    // _.forEach(node.attributes, function(attr) {
    //   var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
    //   if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
    //     normalizedAttrName =
    //       normalizedAttrName[6].toLowerCase() +
    //       normalizedAttrName.substring(7);
    //   }
    //   addDirective(directives, normalizedAttrName);
    // });
    // _.forEach(node.classList, function(cls) {
    //   var normalizedClassName = directiveNormalize(cls);
    //   addDirective(directives, normalizedClassName);
    // });
  } else if (node.nodeType === Node.COMMENT_NODE) {
    
  }
  return directives;
}
```

我们在注释节点分支要做的是，用一个正则表达式来对注释内容进行匹配，检查这个注释是否以`directive`开头。如果是，我们就对匹配项进行标准化，并查找是否有指令与它相匹配。

```js
function collectDirectives(node) {
  var directives = [];
  if (node.nodeType === Node.ELEMENT_NODE) {
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
    _.forEach(node.classList, function(cls) {
      var normalizedClassName = directiveNormalize(cls);
      addDirective(directives, normalizedClassName);
    });
  } else if (node.nodeType === Node.COMMENT_NODE) {
    var match = /^\s*directive\:\s*([\d\w\-_]+)/.exec(node.nodeValue);
    if (match) {
      addDirective(directives, directiveNormalize(match[1]));
    }
  }
  return directives;
}
```

注意，我们所用的表达式是支持在`directive:`前缀的前后加上空格的。
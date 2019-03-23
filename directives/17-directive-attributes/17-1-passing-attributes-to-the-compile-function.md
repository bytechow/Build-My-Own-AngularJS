### 传递属性给 compile 函数（Passing Attributes to the compile Function）

上一章我已经实现了指令对 DOM 编译的支持，还有指令的`compile`函数。我们看到这个函数会接受 jQuery（jqLite）包裹的 DOM 元素作为参数。当然，我们也能够通过 DOM 获取元素上的属性。而实际上，我们会有一种更简单的方法来获取属性，也就是利用 compile 函数的第二参数。第二参数将会一个对象，包含了该元素上的属性，属性名将会用驼峰格式：

_test/compile_spec.js_

```js
describe('attributes', function() {
  
  it('passes the element attributes to the compile function', function() {
    var injector = makeInjectorWithDirectives('myDirective', function() {
      return {
        restrict: 'E',
        compile: function(element, attrs) {
          element.data('givenAttrs', attrs);
        }
      };
    });
    injector.invoke(function($compile) {
      var el = $('<my-directive my-attr="1" my-other-attr="two"></my-directive>');
      $compile(el);

      expect(el.data('givenAttrs').myAttr).toEqual('1');
      expect(el.data('givenAttrs').myOtherAttr).toEqual('two');
    });
  });
});
```

在这个单元测试中，我们希望`compile`函数接收第二个参数，并把这个参数作为节点数据绑定到对应的元素上去。然后我们会检查第二个参数上是否包含绑定在 DOM 元素节点上的指令属性。

除了把名称变成驼峰形式以外，我们还需要做的一件事是去除空格符。在属性值中存在的空格将会被去除：

```js
it('trims attribute values', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      restrict: 'E',
      compile: function(element, attrs) {
        element.data('givenAttrs', attrs);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<my-directive my-attr=" val "></my-directive>');
    $compile(el);
    expect(el.data('givenAttrs').myAttr).toEqual('val');
  });
});
```

下面让我们在 DOM 编译器中加入对上述处理的支持。我们需要为每个正在编译的节点创建一个空对象用于存储属性。我们会在`compileNodes`函数中对节点进行遍历的过程中创建这个对象。创建这个属性对象后，我们会把它陆续传递到`collectDirectives`和`applyDirectivesToNode`里面：

_src/compile.js_

```js
function compileNodes($compileNodes) {
  // _.forEach($compileNodes, function(node) {
    var attrs = {};
    var directives = collectDirectives(node, attrs);
    var terminal = applyDirectivesToNode(directives, node, attrs);
  //   if (!terminal && node.childNodes && node.childNodes.length) {
  //     compileNodes(node.childNodes);
  //   }
  // });
}
```

在`applyDirectivesToNode`方法中，我们会把这个属性对象直接往下传递到指令的 compile 函数中，这样我们就能在测试用例中的 compile 函数中获取到了：

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // _.forEach(directives, function(directive) {
  //   if (directive.$$start) {
  //     $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  //   }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.compile) {
      directive.compile($compileNode, attrs);
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });
  // return terminal;
}
```

收集、存放属性到属性对象的过程会发生在`collectDirectives`中。鉴于我们已经在这个函数中对属性进行了遍历（为了匹配属性式指令），我们可以在这个遍历中顺便把所有的属性都添加到属性对象中：

```js
function collectDirectives(node, attrs) {
  // var directives = [];
  // if (node.nodeType === Node.ELEMENT_NODE) {
  //   var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
  //   addDirective(directives, normalizedNodeName, 'E');
  //   _.forEach(node.attributes, function(attr) {
  //     var attrStartName, attrEndName;
  //     var name = attr.name;
  //     var normalizedAttrName = directiveNormalize(name.toLowerCase());
  //     if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
  //       name = _.kebabCase(
  //         normalizedAttrName[6].toLowerCase() +
  //         normalizedAttrName.substring(7)
  //       );
  //     }
  //     var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
  //     if (directiveIsMultiElement(directiveNName)) {
  //       if (/Start$/.test(normalizedAttrName)) {
  //         attrStartName = name;
  //         attrEndName = name.substring(0, name.length - 5) + 'end';
  //         name = name.substring(0, name.length - 6);
  //       }
  //     }
  //     normalizedAttrName = directiveNormalize(name.toLowerCase());
  //     addDirective(directives,
  //       normalizedAttrName, 'A', attrStartName, attrEndName);
      attrs[normalizedAttrName] = attr.value.trim();
  //   });
  //   _.forEach(node.classList, function(cls) {
  //     var normalizedClassName = directiveNormalize(cls);
  //     addDirective(directives, normalizedClassName, 'C');
  //   });
  // } else if (node.nodeType === Node.COMMENT_NODE) {
  //   var match = /^\s*directive\:\s*([\d\w\-_]+)/.exec(node.nodeValue);
  //   if (match) {
  //     addDirective(directives, directiveNormalize(match[1]), 'M');
  //   }
  // }
  // directives.sort(byPriority);
  // return directives;
}
```


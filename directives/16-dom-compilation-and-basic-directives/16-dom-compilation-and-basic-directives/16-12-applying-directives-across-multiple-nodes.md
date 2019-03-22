### 跨元素应用指令（Applying Directives Across Multiple Nodes）

目前为止，我们看到了指令可以通过4种不同的途径进行匹配。但我们还有一种途径没说，那就是对一连串的兄弟节点应用指令，我们只需要指明要应用范围的开头与结尾即可：

```html
<div my-directive-start>
</div>
<some-other-html></some-other-html>
<div my-directive-end>
</div>
```

但并不是所有指令都能使用这种方式。指令的编写者必须在指令定义对象中显式地设置`multiElement`属性来启用跨元素匹配。而且，这种匹配方式只能使用属性指令。

当我们使用这种属性指令实现跨元素匹配是，我们在指令的`compile`函数得到的 jQuery/jqLite 对象将会包含开始指令元素、结束指令元素及这两个指令之间的所有元素：

_test/compile_spec.js_

```js
it('allows applying a directive to multiple elements', function() {
  var compileEl = false;
  var injector = makeInjectorWithDirectives('myDir', function() {
    return {
      multiElement: true,
      compile: function(element) {
        compileEl = element;
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div my-dir-start></div><span></span><div my-dir-end></div>');
    $compile(el);
    expect(compileEl.length).toBe(3);
  });
});
```

我们会在`collectDirectives`中处理开始指令元素和结束指令元素，具体是在我们在遍历元素属性。我们会在这里检查我们是不是在处理一个多元素指令。检查具体有三步：

1. 如果指令名称以 Start 或者 End 作为后缀结尾，则把这个后缀从指令名称中剔除
2. 然后看看处理后的这个指令名称是否在注册时明确定义为跨元素的指令
3. 确认这个指令是否是跨元素指令的开始指令

如果第2步和第3步的结果都为`true`，那我们就可以认定目前正在处理的是跨元素指令：

```js
_.forEach(node.attributes, function(attr) {
  // var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
  // if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
  //   normalizedAttrName =
  //     normalizedAttrName[6].toLowerCase() +
  //     normalizedAttrName.substring(7);
  // }
  var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
  if (directiveIsMultiElement(directiveNName)) {
    if (/Start$/.test(normalizedAttrName)) {}
  }
  // addDirective(directives, normalizedAttrName, 'A');
});
```

这里有一个新的辅助函数`directiveIsMultiElement`，它会先查看传入的指令名称是否有对应的指令存在。如果存在，它会从注射器中获取这个指令，并查看这个指令的`multiElement`属性值是否被设置为`true`：

```js
function directiveIsMultiElement(name) {
  if (hasDirectives.hasOwnProperty(name)) {
    var directives = $injector.get(name + 'Directive');
    return _.some(directives, {
      multiElement: true
    });
  }
  return false;
}
```

一旦能确认正在处理的指令是一个跨元素指令，那我们就需要存储它的开始和结束元素的属性指令名称，这样方便在后续的编译中获取到所有匹配元素。我们会引入两个局部变量，如果我们遇到了跨元素指令的开始属性就会计算获取相应的开始、结束指令属性名称，就存储到这两个变量里，最后还会把这两个变量都传递给`addDirective`：

```js
_.forEach(node.attributes, function(attr) {
  var attrStartName, attrEndName;
  // var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
  // if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
  //   normalizedAttrName =
  //     normalizedAttrName[6].toLowerCase() +
  //     normalizedAttrName.substring(7);
  // }
  // var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
  // if (directiveIsMultiElement(directiveNName)) {
  //   if (/Start$/.test(normalizedAttrName)) {
      attrStartName = normalizedAttrName;
      attrEndName =
        normalizedAttrName.substring(0, normalizedAttrName.length - 5) + 'End';
      normalizedAttrName =
        normalizedAttrName.substring(0, normalizedAttrName.length - 5);
  //   }
  // }
  addDirective(directives, normalizedAttrName, 'A', attrStartName, attrEndName);
});
```

但是，在我们继续之前还需要修复一个问题。我们现在传递给`addDirective`的是一个经过标准化的、驼峰式的属性名称，这跟我们在 DOM 上使用的已经不完全相同了。这给我们之后根据属性名称来查找 DOM 元素带来麻烦。

我们需要做的是传递一个原始的、未经标准化的属性名称。另外，在 Angular 中还需要处理一种特殊情况，那就是加上`ng-attr-`前缀的开始指令属性，这种属性名称无法被存储。所以我们基本上是不会同时使用`ng-attr-`前缀和`-start`后缀的：

我们会在把可能存在的`ng-attr-`前缀，对属性名称进行"反标准化"，也就是把属性名称变成用连字符的格式，并把这种格式的属性名称来计算、存储开始、结束指令属性名称：

```js
_.forEach(node.attributes, function(attr) {
  // var attrStartName, attrEndName;
  var name = attr.name;
  var normalizedAttrName = directiveNormalize(name.toLowerCase());
  // if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
    name = _.kebabCase(
      normalizedAttrName[6].toLowerCase() +
      normalizedAttrName.substring(7)
    );
  // }
  // var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
  // if (directiveIsMultiElement(directiveNName)) {
  //   if (/Start$/.test(normalizedAttrName)) {
      attrStartName = name;
      attrEndName = name.substring(0, name.length - 5) + 'end';
      name = name.substring(0, name.length - 6);
  //   }
  // }
  normalizedAttrName = directiveNormalize(name.toLowerCase());
  // addDirective(directives, normalizedAttrName, 'A', attrStartName, attrEndName);
});
```

现在我们的`collectDirectives`算是开发得差不多了，接下来我们来看看`addDirective`方法，上面我们可以看到，这个方法现在已经多出来两个（可选）参数：开始指令名称和结束指令名称。

在`addDirective`中，如果发现传入了开始指令和结束指令名称，我们会把它们作为特殊属性`$$start`和`$$end`的值，参与到指令对象的创建过程来。我们使用`$$`这种特殊前缀，是因为我们不想污染源阿里的指令对象。我们可能多次使用一个指令，而这个指令可能带上开始和结束标志，也可能没有，因此采用这种扩展标记显得非常必要:

```js
function addDirective(directives, name, mode, attrStartName, attrEndName) {
  // if (hasDirectives.hasOwnProperty(name)) {
  //   var foundDirectives = $injector.get(name + 'Directive');
  //   var applicableDirectives = _.flter(foundDirectives, function(dir) {
  //     return dir.restrict.indexOf(mode) !== -1;
  //   });
    _.forEach(applicableDirectives, function(directive) {
      if (attrStartName) {
        directive = _.create(directive, {
          $$start: attrStartName,
          $$end: attrEndName
        });
      }
      directives.push(directive);
    });
  // }
}
```

下一步，我们需要在`applyDirectivesToNode`进行，这里也是我们真正调用指令的compile方法进行编译的地方。在我们进行编译指令之前，我们需要看看这个指令是否包含开始指令标志，如果是的话，需要我们把开始指令元素、结束指令元素以及它们之间的所有元素作为待编译的元素节点集。我们希望把这个获取元素节点集的过程放到一个叫`groupScan`的函数里面去：

```js
function applyDirectivesToNode(directives, compileNode) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // _.forEach(directives, function(directive) {
    if (directive.$$start) {
      $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
    }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.compile) {
  //     directive.compile($compileNode);
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });
  // return terminal;
}
```

这个新函数将会接收三个参数：一个作为检索起点的元素节点、一个开始指令属性名和一个结束指令属性名：

```js
function groupScan(node, startAttr, endAttr) {

}
```

这个函数将会检查这个起始节点是否包含有开始指令属性。如果不符合这个特殊条件，这个函数就会把这个起始节点作为唯一的匹配元素返回回去：

```js
function groupScan(node, startAttr, endAttr) {
  var nodes = [];
  if (startAttr && node && node.hasAttribute(startAttr)) {} else {
    nodes.push(node);
  }
  return $(nodes);
}
```

繁殖，如果起始节点有开始指令属性，我们就会开始遍历收集元素节点集。在这个过程中，我们需要借助一个局部变量`depth`，当深度达到0的时候，我们就终止遍历过程：

```js
function groupScan(node, startAttr, endAttr) {
  // var nodes = [];
  // if (startAttr && node && node.hasAttribute(startAttr)) {
    var depth = 0;
    do {
      
    } while (depth > 0);
  // } else {
  //   nodes.push(node);
  // }
  // return $(nodes);
}
```

在循环中，我们会对节点的相邻节点进行遍历，并把找到的节点都加入数组`nodes`中存储起来，成为元素节点集的一个元素。

```js
function groupScan(node, startAttr, endAttr) {
  // var nodes = [];
  // if (startAttr && node && node.hasAttribute(startAttr)) {
  //   var depth = 0;
  //   do {
      nodes.push(node);
      node = node.nextSibling;
  //   } while (depth > 0);
  // } else {
  //   nodes.push(node);
  // }
  // return $(nodes);
}
```

这个版本我们收集了所有相邻节点，但我们只需要收集到结束指令元素就可以了。在这里，`depth`变量就起到作用了。当我看到一个元素含有开始指令属性，我们就把深度标识加1，而如果遇到对应的结束指令属性，我们就把深度标识减1。这就是说，只要我们检查到结束标识与开始标识一样多是，我们就结束掉循环语句。

```js
function groupScan(node, startAttr, endAttr) {
  // var nodes = [];
  // if (startAttr && node && node.hasAttribute(startAttr)) {
  //   var depth = 0;
  //   do {
      if (node.nodeType === Node.ELEMENT_NODE) {
        if (node.hasAttribute(startAttr)) {
          depth++;
        } else if (node.hasAttribute(endAttr)) {
          depth--;
        }
      }
  //     nodes.push(node);
  //     node = node.nextSibling;
  //   } while (depth > 0);
  // } else {
  //   nodes.push(node);
  // }
  // return $(nodes);
}
```

大部分情况下，深度只会到1，但我们这样的实现方式可以支持嵌套的指令组。就像下面的 DOM 结构一样，我们需要收集到外层的指令组，就需要收集它到第二个`my-dir-end`之间所有的元素，它的深度会到达2:

```html
<div my-dir-start></div>
<div my-dir-start></div>
<div my-dir-end></div>
<div my-dir-end></div>
```

这样我们就可以跨元素使用指令了！
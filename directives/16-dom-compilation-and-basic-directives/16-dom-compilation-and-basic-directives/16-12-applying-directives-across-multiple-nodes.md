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
  var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
  if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
    normalizedAttrName =
      normalizedAttrName[6].toLowerCase() +
      normalizedAttrName.substring(7);
  }
  var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
  if (directiveIsMultiElement(directiveNName)) {
    if (/Start$/.test(normalizedAttrName)) {
      attrStartName = normalizedAttrName;
      attrEndName =
        normalizedAttrName.substring(0, normalizedAttrName.length - 5) + 'End';
      normalizedAttrName =
        normalizedAttrName.substring(0, normalizedAttrName.length - 5);
    }
  }
  addDirective(directives, normalizedAttrName, 'A', attrStartName, attrEndName);
});
```
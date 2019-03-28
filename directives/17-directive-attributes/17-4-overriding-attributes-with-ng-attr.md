### 使用 ng-attr 覆盖属性（Overriding attributes with ng-attr）

我们已经知道可以用`ng-attr-`作为 Angular 指令属性的前缀，而这个前缀会在属性被收集期间去除。但如果我们同时对一个元素加上两个同名属性，一个使用了`ng-attr-`前缀，一个没用，会发生什么事情？对于目前已实现的代码，这种情况产生的结果取决于两者谁先被声明，但 Angular 实际上并不会依赖声明顺序：带上`ng-attr-`前缀的属性将会覆盖没带上的。

_test/compile\_spec.js_

```js
it('overrides attributes with ng-attr- versions', function() {
  registerAndCompile(
    'myDirective',
    '<input my-directive ng-attr-whatever="42" whatever="41">',
    function(element, attrs) {
      expect(attrs.whatever).toEqual('42');
    }
  );
});
```

当我们遍历属性时，我们会对带有`ng-attr-`前缀的属性做一个布尔值标记。之前当我们保存属性时，我们会先检查这个属性是否已经被保存。现在只要这个属性是带有`ng-attr-`前缀的话，我们就会保存这个值，无论这个属性之前是否已经存在，这就可以满足覆盖属性的功能：

_src/compile.js_

```js
function collectDirectives(node, attrs) {
  var directives = [];
  if (node.nodeType === Node.ELEMENT_NODE) {
    var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
    addDirective(directives, normalizedNodeName, 'E');
    _.forEach(node.attributes, function(attr) {
      var attrStartName, attrEndName;
      var name = attr.name;
      var normalizedAttrName = directiveNormalize(name.toLowerCase());
      var isNgAttr = /^ngAttr[A-Z]/.test(normalizedAttrName);
      if (isNgAttr) {
      //   name = _.kebabCase(
      //     normalizedAttrName[6].toLowerCase() +
      //     normalizedAttrName.substring(7)
      //   );
      // }
      // var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
      // if (directiveIsMultiElement(directiveNName)) {
      //   if (/Start$/.test(normalizedAttrName)) {
      //     attrStartName = name;
      //     attrEndName = name.substring(0, name.length - 5) + 'end';
      //     name = name.substring(0, name.length - 6);
      //   }
      // }
      // normalizedAttrName = directiveNormalize(name.toLowerCase());
      // addDirective(directives, normalizedAttrName, 'A',
      //   attrStartName, attrEndName);
      if (isNgAttr || !attrs.hasOwnProperty(normalizedAttrName)) {
        // attrs[normalizedAttrName] = attr.value.trim();
        // if (isBooleanAttribute(node, normalizedAttrName)) {
        //   attrs[normalizedAttrName] = true;
        // }
      }
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




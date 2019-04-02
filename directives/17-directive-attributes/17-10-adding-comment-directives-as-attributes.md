### 把注释式指令添加到指令属性中（Adding Comment Directives As Attributes）

就像样式类式的指令一样，注释式指令也能添加到属性对象中，同时也可以关联一个属性值：

_test/compile_spec.js_

```js
it('adds an attribute with a value from a comment directive', function() {
  registerAndCompile(
    'myDirective',
    '<!-- directive: my-directive and the attribute value -->',
    function(element, attrs) {
      expect(attrs.hasOwnProperty('myDirective')).toBe(true);
      expect(attrs.myDirective).toEqual('and the attribute value');
    }
  );
});
```

注释式指令比样式类式指令更容易处理，因为每个注释节点中很可能只有一个指令。不过，与样式类样式一样，我们仍然需要使用正则表达式。我们之前在解析注释类指令时已经有一个正则表达式。这个正则表达式在匹配指令名称之后，会允许有一些空白符作为分割，之后我们会把剩余的注释内容放到一个捕获组里面：

```js
// } else 
// if (node.nodeType === Node.COMMENT_NODE) {
  match = /^\s*directive\:\s*([\d\w\-_]+)\s*(.*)$/.exec(node.nodeValue);
  // if (match) {
    var normalizedName = directiveNormalize(match[1]);
    if (addDirective(directives, normalizedName, 'M')) {
      attrs[normalizedName] = match[2] ? match[2].trim() : undefned;
    }
//   }
// }
```

这个匹配模式其实就跟样式类指令一样了：如果在注释中匹配到一个指令，这个标准化后的指令名称就会添加到属性对象上。
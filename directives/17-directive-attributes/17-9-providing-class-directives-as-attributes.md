### 把样式类式指令也添加到指令属性中（Providing Class Directives As Attributes）

上面我们讨论了 Angular 可以通过`Attributes`对象访问元素上的属性。但`Attributes`对象中不仅仅可以包含元素属性。样式类式和注释式指令在某些情况下也可能作为指令属性进行保存。

首先，如果存在一个样式类式的指令，那它的名称也会在属性对象中出现：

_test/compile_spec.js_

```js
t('adds an attribute from a class directive', function() {
  registerAndCompile(
    'myDirective',
    '<div class="my-directive"></div>',
    function(element, attrs) {
      expect(attrs.hasOwnProperty('myDirective')).toBe(true);
    }
  );
});
````

在`collectDirectives`方法中，我们会把这种指令名称也放到属性对象里面：

_src/compile.js_

```js
_.forEach(node.classList, function(cls) {
  var normalizedClassName = directiveNormalize(cls);
  addDirective(directives, normalizedClassName, 'C');
  attrs[normalizedClassName] = undefined;
});
```

这个属性的值是`undefined`，但仍然会在属性对象中存在，所以上面测试单元中的`hasOwnProperty`检查会返回`true`。

但并不是所有样式类都可以作为指令属性存放到属性对象中。只有在样式类与某个指令名称相匹配我们才会这样做，而目前的实现却把所有样式类添加进去了：

_test/compile_spec.js_

```js
it('does not add attribute from class without a directive', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive class="some-class"></my-directive>',
    function(element, attrs) {
      expect(attrs.hasOwnProperty('someClass')).toBe(false);
    }
  );
});
```

在我们把样式类指令添加到属性对象之前，我们需要先查看是否有指令跟这个样式类匹配。我们可以在`addDirective`函数中把匹配结果进行返回：

_src/compile.js_

```js
function addDirective(directives, name, mode, attrStartName, attrEndName) {
  var match;
  if (hasDirectives.hasOwnProperty(name)) {
    var foundDirectives = $injector.get(name + 'Directive');
    var applicableDirectives = _.flter(foundDirectives, function(dir) {
      return dir.restrict.indexOf(mode) !== -1;
    });
    _.forEach(applicableDirectives, function(directive) {
      if (attrStartName) {
        directive = _.create(directive, {
          $$start: attrStartName,
          $$end: attrEndName
        });
      }
      directives.push(directive);
      match = directive;
    });
  }
  return match;
}
```

函数返回的具体值要么是`undefined`，要么是一个匹配到的指令定义对象，而我们的测试用例目前只检查返回值是否一个真值。

看来样式类式指令的返回值就是`undefined`了，其实也不一定。你可以通过在样式类指令名称后加上冒号的方式来添加属性值，这个值甚至可以带上空格符。

_test/compile_spec.js_

```js
it('supports values for class directive attributes', function() {
  registerAndCompile(
    'myDirective',
    '<div class="my-directive: my attribute value"></div>',
    function(element, attrs) {
      expect(attrs.myDirective).toEqual('my attribute value');
    }
  );
});
```

这类属性值默认会把冒号后面的内容都作为指令的属性值，但也可以通过添加分号来提前终止。在分号之后就可以添加其他 CSS 样式类，这些样式类既可以是与指令相关的，也可能是无关的：

```js
it('terminates class directive attribute value at semicolon', function() {
  registerAndCompile(
    'myDirective',
    '<div class="my-directive: my attribute value; some-other-class"></div>',
    function(element, attrs) {
      expect(attrs.myDirective).toEqual('my attribute value');
    }
  );
});
```

在前面的章节中国年，我们是直接使用`classList`数组来获取元素 DOM 上的 CSS 样式类。为了支持对样式类属性值中的指令提取，我们需要换成使用`className` API 来获取样式类字符串，然后使用正则表达式匹配。

首先，我们会对`collectDirectives`方法进行重新组织，我们需要一个本地变量来存放正则表达式匹配到的内容。这个变量我们同样命名为`match`。我们在注释式指令匹配时也需要使用`match`变量，所以这里我们会把这个局部变量提升到到`collectDirectives`方法作用域的顶层。然后我们会用对样式类字符串的处理替代对`classList`的遍历，而这也只会在`className`属性不为空时候发生：

_src/compile.js_

```js
function collectDirectives(node, attrs) {
  // var directives = [];
  var match;
  // if (node.nodeType === Node.ELEMENT_NODE) {
  //   var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
  //   addDirective(directives, normalizedNodeName, 'E');
  //   _.forEach(node.attributes, function(attr) {
  //     var attrStartName, attrEndName;
  //     var name = attr.name;
  //     var normalizedAttrName = directiveNormalize(name.toLowerCase());
  //     var isNgAttr = /^ngAttr[A-Z]/.test(normalizedAttrName);
  //     if (isNgAttr) {
  //       name = _.kebabCase(
  //         normalizedAttrName[6].toLowerCase() + normalizedAttrName.substring(7)
  //       );
  //       normalizedAttrName = directiveNormalize(name.toLowerCase());
  //     }
  //     attrs.$attr[normalizedAttrName] = name;
  //     var directiveNName = normalizedAttrName.replace(/(Start|End)$/, '');
  //     if (directiveIsMultiElement(directiveNName)) {
  //       if (/Start$/.test(normalizedAttrName)) {
  //         attrStartName = name;
  //         attrEndName = name.substring(0, name.length - 5) + 'end';
  //         name = name.substring(0, name.length - 6);
  //       }
  //     }
  //     normalizedAttrName = directiveNormalize(name.toLowerCase());
  //     addDirective(
  //       directives, normalizedAttrName, 'A', attrStartName, attrEndName);
  //     if (isNgAttr || !attrs.hasOwnProperty(normalizedAttrName)) {
  //       attrs[normalizedAttrName] = attr.value.trim();
  //       if (isBooleanAttribute(node, normalizedAttrName)) {
  //         attrs[normalizedAttrName] = true;
  //       }
  //     }
  //   });
    var className = node.className;
    if (_.isString(className) && !_.isEmpty(className)) {
      
    }
  // } else if (node.nodeType === Node.COMMENT_NODE) {
    match = /^\s*directive\:\s*([\d\w\-_]+)/.exec(node.nodeValue);
    if (match) {
      addDirective(directives, directiveNormalize(match[1]), 'M');
    }
  // }
  // directives.sort(byPriority);
  // return directives;
}
```

你可能猜到了，就像我们使用`$injector`对函数字符串进行解析一样，这里也需要做一些正则表达式的准备工作，我们需要精心准备一个正则表达式才行：

- 匹配样式类名称，他们之间会使用空格分割
- 如果样式类名称后面带上冒号，就准备匹配这个样式类属性的值。这个值可能会包含空格符。
- 当遇到分号时，就停止当前样式类的匹配，继续匹配下一个样式类。

我这里直接列出我们要用的正则表达式：

```js
/([\d\w\-_]+)(?:\:([^;]+))?;?/
```

- `([\d\w\-_]+)`将匹配并捕获一个由数字、字母、连字符和下划线组成的字符串。这就是用于匹配一个样式类名。
- `(?:`表示一个非捕获组的开始，后面的`)?`代表非捕获组的结束，并表明这个捕获组里面的内容是可选的。而最后的`;?`表示最后的分号结尾也是可选的。
- 在非捕获组中，`\:`匹配一个冒号，而`([^;]+)`匹配多个不是分号的字符，并作为本个匹配表达式的第二个捕获组。这就是对属性值的捕获。

如果我们在处理样式类名称时使用该正则表达式，我们可以对样式名称进行“消费”，每一个循环都会找出一个样式名称、或是一个带上值的样式名称：

_src/compile.js_

```js
className = node.className;
if (_.isString(className) && !_.isEmpty(className)) {
  while ((match = /([\d\w\-_]+)(?:\:([^;]+))?;?/.exec(className))) {
    className = className.substr(match.index + match[0].length);
  }
}
```

为了避免无限循环，在每次循环中我们都需要去除`className`已匹配的部分，并用剩余部分来更新`className`的值。我们会利用匹配后的索引值和字符串`substr`来完成截取工作。

现在我们在执行样式类字符串检索的循环中不仅可以找到指令，还可以得到指令属性和它的值：

```js
className = node.className;
if (isString(className) && className !== '') {
  while ((match = /([\d\w\-_]+)(?:\:([^;]+))?;?/.exec(className))) {
    var normalizedClassName = directiveNormalize(match[1]);
    if (addDirective(directives, normalizedClassName, 'C')) {
      attrs[normalizedClassName] = match[2] ? match[2].trim() : undefned;
    }
    className = className.substr(match.index + match[0].length);
  }
}
```

指令名称将会是匹配组的第二个元素（因为它被第一个捕获组捕获）。而对于属性值，如果它存在的话，就会是匹配组的第三个元素。

有了这套匹配系统，现在我们样式类的新旧单元测试都能通过了。
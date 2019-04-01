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

首先，我们会对`collectDirectives`方法进行重新组织，我们需要一个本地变量来存放正则表达式匹配到的内容。这个变量我们同样命名为`match`。我们在前面指令匹配时使用过一个同名变量，所以这里我们会把这个局部变量放到`collectDirectives`方法的顶层。
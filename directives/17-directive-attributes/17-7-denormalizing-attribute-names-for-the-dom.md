### 对 DOM 上的特性名称反向标准化（Denormalizing Attribute Names for The DOM）

在属性对象中，我们使用的是经过标准化后的属性名称。这种指令名称实际上并不是它们在 DOM 中的样子，但确实更方便 JavaScript 操作。特别是，我们会使用驼峰形式的属性名称而不是使用连字符形式的属性名称。另外，正如我们在前面的章节看到的，属性可能加上的几种特殊前缀也会被去除。

当我们需要设置 DOM 属性时，这种标准化后的属性名就会成为一个问题。我们无法在 DOM 上使用这种标准化后的属性名，因为结果可能并不是我们想象那么顺利（除了像`'attr'`这样简单的名字）。当我们需要在 DOM 上更新属性时，我们需要对属性名称进行反向标准化。

要完成反向标准化，可以有几种处理方案。最直接的方法就是使用`$set`的第四个参数来直接指明一个反向标准化的属性名称：

_test/compile_spec.js_

```js
it('denormalizes attribute name when explicitly given', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive some-attribute="42"></my-directive>',
    function(element, attrs) {
      attrs.$set('someAttribute', 43, true, 'some-attribute');
      expect(element.attr('some-attribute')).toEqual('43');
    }
  );
});
```

在`$set`方法中我们默认使用第四个参数作为之后要进行的属性名称，如果调用时没传入第四个参数，我们就会像之前那样使用第一个参数：

_src/compile.js_

```js
Attributes.prototype.$set = function(key, value, writeAttr, attrName) {
  // this[key] = value;

  // if (isBooleanAttribute(this.$$element[0], key)) {
  //   this.$$element.prop(key, value);
  // }
  
  if (!attrName) {
    attrName = key;
  }
  // if (writeAttr !== false) {
    this.$$element.attr(attrName, value);
  // }
};
```

这种处理方式会起效，但对使用者并不友好。调用`$set`时需要传入两个版本的属性名称，这并不是最理想的。

另一种处理方案是，当我们没有明确指明时，我们会默认把属性名变成连字符形式的：

_test/compile_spec.js_

```js
it('denormalizes attribute by snake-casing', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive some-attribute="42"></my-directive>',
    function(element, attrs) {
      attrs.$set('someAttribute', 43);
      expect(element.attr('some-attribute')).toEqual('43');
    }
  );
});
```

这里我们可以直接使用 LoDash 提供的`_.kebabCase`方法：

_src/compile.js_

```js
Attributes.prototype.$set = function(key, value, writeAttr, attrName) {
  this[key] = value;

  if (isBooleanAttribute(this.$$element[0], key)) {
    this.$$element.prop(key, value);
  }

  if (!attrName) {
    attrName = _.kebabCase(key, '-');
  }

  if (writeAttr !== false) {
    this.$$element.attr(attrName, value);
  }
};
```

但这种解决方案还不是最理想的。正如我们之前讨论的，Angular 支持在 DOM 属性上加入几种前缀，这些前缀会在标准化时去除。你会希望用`$set`时能把整个 DOM 属性都被更新，包括前缀。但`_.kebabCase`并不能包含这些前缀。我们应该对前缀也进行支持：

_test/compile_spec.js_

```js
it('denormalizes attribute by using original attribute name', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive x-some-attribute="42"></my-directive>',
    function(element, attrs) {
      attrs.$set('someAttribute', '43');
      expect(element.attr('x-some-attribute')).toEqual('43');
    }
  );
});
```

唯一例外的前缀就是`ng-attr-`，在我们使用`$set`设置带有这种前缀的属性时，要更新的属性名称并不会包含这个前缀：

```js
it('does not use ng-attr- prefx in denormalized names', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive ng-attr-some-attribute="42"></my-directive>',
    function(element, attrs) {
      attrs.$set('someAttribute', 43);
      expect(element.attr('some-attribute')).toEqual('43');
    }
  );
});
```

我们需要在标准化之前，就使用一个对象来存储原始的属性名称和对应的属性名称的映射关系。这个映射会被保存在`Attributes`实例的`$attr`属性中：

_src/compie.js_

```js
function Attributes(element) {
  this.$$element = element;
  this.$attr = {};
}
```

在`$set`方法中，如果没有显式地传入属性名称（第四个参数），我们会先在`$attr`对象中查找这个属性的 key。当在映射中无法找到这个 key 时，我们才会使用`_.kebabCase`把 key 变成连字符形式，并把它作为属性名称保存起来，方便以后调用`$set`：

```js
Attributes.prototype.$set = function(key, value, writeAttr, attrName) {
  // this[key] = value;
  
  // if (isBooleanAttribute(this.$$element[0], key)) {
  //   this.$$element.prop(key, value);
  // }

  // if (!attrName) {
    if (this.$attr[key]) {
      attrName = this.$attr[key];
    } else {
      attrName = this.$attr[key] = _.kebabCase(key);
    }
  // }
  
  // if (writeAttr !== false) {
  //   this.$$element.attr(attrName, value);
  // }
};
```

这个映射对象会在`collectDirectives`方法中进行赋值。这个方法能够访问属性对象，它会直接使用属性对象的`$attr`属性保存标准化-非标准化属性名称的映射。这个过程会发生在`_.forEach(node.attributes)`循环中，在处理完`ng-attr-`属性之后：

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
        name = _.kebabCase(
          normalizedAttrName[6].toLowerCase() +
          normalizedAttrName.substring(7)
        );
        normalizedAttrName = directiveNormalize(name.toLowerCase());
      }
      attrs.$attr[normalizedAttrName] = name;

      // ...
    
    });
    
    // ...
  
  } // ...
}
```

最后一点，当我们在调用`$set`方法时显式地传入第四个参数后，在映射中，这个标准化名称对应的 DOM 属性名也会替换为这个参数名。也就是说，之后再次使用这个标准化名称进行`$set`时，只会更新指定的属性，而不会更新原始的、非标准化的名称：

_test/compile_spec.js_

```js
it('uses new attribute name after once given', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive x-some-attribute="42"></my-directive>',
    function(element, attrs) {
      attrs.$set('someAttribute', 43, true, 'some-attribute');
      attrs.$set('someAttribute', 44);

      expect(element.attr('some-attribute')).toEqual('44');
      expect(element.attr('x-some-attribute')).toEqual('42');
    }
  );
});
```

所以，如果在`$set`方法中传入`attrName`参数，也会更新`$attr`对象：

_src/compile.js_

```js
Attributes.prototype.$set = function(key, value, writeAttr, attrName) {
  this[key] = value;

  if (isBooleanAttribute(this.$$element[0], key)) {
    this.$$element.prop(key, value);
  }

  if (!attrName) {
    if (this.$attr[key]) {
      attrName = this.$attr[key];
    } else {
      attrName = this.$attr[key] = _.kebabCase(key);
    }
  } else {
    this.$attr[key] = attrName;
  }
  
  if (writeAttr !== false) {
    this.$$element.attr(attrName, value);
  }
};
```
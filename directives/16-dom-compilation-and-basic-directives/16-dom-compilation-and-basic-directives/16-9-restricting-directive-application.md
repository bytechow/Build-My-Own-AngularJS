### 限定指令应用类型（Restricting Directive Application）

目前，我们知道 Angular 有四种途径可以在 DOM 中应用指令：HTML 元素、HTML 属性、样式类名和特殊格式的注释。

但这并不是代表只要开发者写了合法的、匹配的指令名称，就能够使用这个指令的。指令的编写者可以对指令的应用类型进行限制，规定指令只能使用四种应用类型中的一个或多个。这个特性有用之处在于，举例说，这样我们不会让某个实现了自定义元素的指令被意外应用到注释中去。有了这种机制，就能使这种情况下即使注释格式匹配了，也不会应用上指令。

我们可以在指令定义对象中加入`restrict`这个属性来实现指令应用类型的限制。这个属性的值由最多四个大写字母组成的特定域名语言（domain-specific language）组成，下面我们列出了一些示例值：

- `E`代表“只允许通过元素名进行匹配”
- `A`代表“只允许通过属性名进行匹配”
- `C`代表“只允许通过样式类名进行匹配”
- `M`代表“只允许通过注释进行匹配”
- `EA`代表“允许通过元素名称和属性进行匹配”
- `MCA`代表“允许通过注释、样式雷鸣和属性进行匹配”
- 其他可能的组合

我们会用一些“生产力”更高的测试方法，这样我们就不必为各种可能出现的情况编写大量的测试用例代码。我们会用对象作为一个特殊的数据结构来存储我们的输入和输出。输入是 restrict 的各种组合，而输出会用一个对象来明确我们需要得出的结果。然后，我们会对这个对象进行遍历，以生成各个`describe`块：

_test/compile_spec.js_

```js
_.forEach({
  E: {element: true, attribute: false, class: false, comment: false},
  A: {element: false, attribute: true, class: false, comment: false},
  C: {element: false, attribute: false, class: true, comment: false},
  M: {element: false, attribute: false, class: false, comment: true},
  EA: {element: true, attribute: true, class: false, comment: false},
  AC: {element: false, attribute: true, class: true, comment: false},
  EAM: {element: true, attribute: true, class: false, comment: true},
  EACM: {element: true, attribute: true, class: true, comment: true},
}, function(expected, restrict) {

  describe('restricted to '+restrict, function() {

  });
  
});
```

在每个循环中，我们再用一个循环来检测当前可以用于注册指令的各种 DOM 写法：

```js
_.forEach({
  E: {element: true, attribute: false, class: false, comment: false},
  A: {element: false, attribute: true, class: false, comment: false},
  C: {element: false, attribute: false, class: true, comment: false},
  M: {element: false, attribute: false, class: false, comment: true},
  EA: {element: true, attribute: true, class: false, comment: false},
  AC: {element: false, attribute: true, class: true, comment: false},
  EAM: {element: true, attribute: true, class: false, comment: true},
  EACM: {element: true, attribute: true, class: true, comment: true},
}, function(expected, restrict) {

  describe('restricted to ' + restrict, function() {

    _.forEach({
      element: '<my-directive></my-directive>',
      attribute: '<div my-directive></div>',
      class: '<div class="my-directive"></div>',
      comment: '<!-- directive: my-directive -->'
    }, function(dom, type) {
      it((expected[type] ? 'matches' : 'does not match') + ' on ' + type, function() {
        var hasCompiled = false;
        var injector = makeInjectorWithDirectives('myDirective', function() {
          return {
            restrict: restrict,
            compile: function(element) {
              hasCompiled = true;
            }
          };
        });
        injector.invoke(function($compile) {
          var el = $(dom);
          $compile(el);
          expect(hasCompiled).toBe(expected[type]);
        });
      });

    });

  });
  
})
```

这样我就生成了32个新的单元测试。如果你想要增加更多组合的测试，也十分简单。

对指令的限制会在`addDirectives`函数中被实现。在此之前，我们需要对`collectDirectives`进行修改，让`addDirectives`方法能获知要用哪种匹配限制模式：

_src/compile.js_

```js
function collectDirectives(node) {
  // var directives = [];
  // if (node.nodeType === Node.ELEMENT_NODE) {
  //   var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
    addDirective(directives, normalizedNodeName, 'E');
    // _.forEach(node.attributes, function(attr) {
    //   var normalizedAttrName = directiveNormalize(attr.name.toLowerCase());
    //   if (/^ngAttr[A-Z]/.test(normalizedAttrName)) {
    //     normalizedAttrName =
    //       normalizedAttrName[6].toLowerCase() +
    //       normalizedAttrName.substring(7);
    //   }
      addDirective(directives, normalizedAttrName, 'A');
    // });
    // _.forEach(node.classList, function(cls) {
    //   var normalizedClassName = directiveNormalize(cls);
      addDirective(directives, normalizedClassName, 'C');
    // });
  // } else if (node.nodeType === Node.COMMENT_NODE) {
  //   var match = /^\s*directive\:\s*([\d\w\-_]+)/.exec(node.nodeValue);
  //   if (match) {
      addDirective(directives, directiveNormalize(match[1]), 'M');
    // }
  // }
  // return directives;
}
```

`addDirective`函数可以过滤出符合当前指令匹配限制模式（restrict属性）的指令：

```js
function addDirective(directives, name, mode) {
  if (hasDirectives.hasOwnProperty(name)) {
    var foundDirectives = $injector.get(name + 'Directive');
    var applicableDirectives = _.flter(foundDirectives, function(dir) {
      return dir.restrict.indexOf(mode) !== -1;
    });
    directives.push.apply(directives, applicableDirectives);
  }
}
```

当我们在代码中加入上面的变更后，我们会发现一个不幸的后果：我们之前编写与指令相关的测试用例都将出错。这是之前的单元测试都没有加上`restrict`属性，而这个属性现在已经变成我们必须要加上的。要使它们恢复正常，我们需要把从`‘compiles element directives from a single element’`开始的所有单元测试的指令定义对象中加入值为`‘EACM’`的`restrict`属性：

_test/compile_spec.js_

```js
it('compiles element directives from a single element', function() {
  // var injector = makeInjectorWithDirectives('myDirective', function() {
  //   return {
      restrict: 'EACM',
  //     compile: function(element) {
  //       element.data('hasCompiled', true);
  //     }
  //   };
  // });
  // injector.invoke(function($compile) {
  //   var el = $('<my-directive></my-directive>');
  //   $compile(el);
  //   expect(el.data('hasCompiled')).toBe(true);
  // });
});
```

最后，其实`restrict`属性会有一个默认值`EA`。也就是说，如果我们没有定义`restrict`属性，那这个指令就只会用元素名称或属性名称进行匹配：

```js
it('applies to attributes when no restrict given', function() {
  var hasCompiled = false;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        hasCompiled = true;
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive></div>');
    $compile(el);
    expect(hasCompiled).toBe(true);
  });
});

it('applies to elements when no restrict given', function() {
  var hasCompiled = false;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        hasCompiled = true;
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<my-directive></my-directive>');
    $compile(el)
    expect(hasCompiled).toBe(true);
  });
});

it('does not apply to classes when no restrict given', function() {
  var hasCompiled = false;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        hasCompiled = true;
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div class="my-directive"></div>');
    $compile(el);
    expect(hasCompiled).toBe(false);
  });
});
```

我们会在指令的 factory 函数中加入默认值，在这里我们第一次获取到指令函数：

_src/compile.js_

```js
this.directive = function(name, directiveFactory) {
  // if (_.isString(name)) {
  //   if (name === 'hasOwnProperty') {
  //     throw 'hasOwnProperty is not a valid directive name';
  //   }
  //   if (!hasDirectives.hasOwnProperty(name)) {
  //     hasDirectives[name] = [];
  //     $provide.factory(name + 'Directive', ['$injector', function($injector) {
  //       var factories = hasDirectives[name];
        return _.map(factories, function(factory) {
          var directive = $injector.invoke(factory);
          directive.restrict = directive.restrict || 'EA';
          return directive;
        });
  //     }]);
  //   }
  //   hasDirectives[name].push(directiveFactory);
  // } else {
  //   _.forEach(name, _.bind(function(directiveFactory, name) {
  //     this.directive(name, directiveFactory);
  //   }, this));
  // }
};
```


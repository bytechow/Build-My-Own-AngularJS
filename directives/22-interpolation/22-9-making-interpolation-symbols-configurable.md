### 自定义 interpolation 标识（Making Interpolation Symbols Configurable）

我们离完成`$interpolate`服务这个目标还差一个东西，也就是自定义 interpolation 表达式开启和结束的标识。也就是说，Angular 时允许你把默认的`{{`和`}}`改成其他你希望的标识。

这些功能的价值是被人质疑的，很大程度上因为改了标识之后，会让你的应用程序视图不同于世界上的其他 Angular 应用程序。但既然有这项功能，我们还是了解一下它是怎么实现的好。

在配置阶段，我们可以通过调用`$interpolateProvider`上的两个方法——`startSymbol`和`endSymbol`来分别改变开始和结束标志。而在运行阶段，我们也可以通过`$interpolate`服务本身的`startSymbol`和`endSymbol`两个方法获知（但不能改变）这些标识：

_test/interpolate_spec.js_

```js
it('allows configuring start and end symbols', function() {
  var injector = createInjector(['ng', function($interpolateProvider) {
    $interpolateProvider.startSymbol('FOO').endSymbol('OOF');
  }]);
  var $interpolate = injector.get('$interpolate');
  expect($interpolate.startSymbol()).toEqual('FOO');
  expect($interpolate.endSymbol()).toEqual('OOF');
});
```

在 provider 中，我们会设置这两个方法作为 setter，同时，如果我们调用方法时不带上参数，这两个方法就会变成 getter。在 provider 中，这两个标识会存储在变量中：

_src/interpolate.js_

```js
function $InterpolateProvider() {
  var startSymbol = '{{';
  var endSymbol = '}}';
  this.startSymbol = function(value) {
    if (value) {
      startSymbol = value;
      return this;
    } else {
      return startSymbol;
    }
  };
  this.endSymbol = function(value) {
    if (value) {
      endSymbol = value;
      return this;
    } else {
      return endSymbol;
    }
  };
  // ...
}
```

而在运行阶段，当有了 getter 时，我们就把它添加到`$interpolate`函数上。由于运行阶段不允许改变标识，所以我们使用`_.constant`直接返回变量即可：

```js
function $InterpolateProvider() {
  var startSymbol = '{{';
  var endSymbol = '}}';
  
  // ...
  
  this.$get = ['$parse', function($parse) {
    
    function $interpolate(text, mustHaveExpressions) {
    
      // ...
    
    }
    
    $interpolate.startSymbol = _.constant(startSymbol);
    $interpolate.endSymbol = _.constant(endSymbol);
    
    return $interpolate;
  }];
}
```

现在我们可以自定义标识，就来用用看效果如何吧。下面这个测试配置两个标识分别为`FOO`和`OOF`，然后看看它们运行起来是否保有原有的功能：

_test/interpolate_spec.js_

```js
it('works with start and end symbols that differ from default', function() {
  var injector = createInjector(['ng', function($interpolateProvider) {
    $interpolateProvider.startSymbol('FOO').endSymbol('OOF');
  }]);
  var $interpolate = injector.get('$interpolate');
  var interpFn = $interpolate('FOOmyExprOOF');
  expect(interpFn({myExpr: 42})).toEqual('42');
});
```

另外，如果改变了标识，我们应该检查一下默认的标识是否已经失效。它们应该被翻译为静态文本：

```js
it('does not work with default symbols when reconfigured', function() {
  var injector = createInjector(['ng', function($interpolateProvider) {
    $interpolateProvider.startSymbol('FOO').endSymbol('OOF');
  }]);
  var $interpolate = injector.get('$interpolate');
  var interpFn = $interpolate('{{myExpr}}');
  expect(interpFn({myExpr: 42})).toEqual('{{myExpr}}');
});
```

解决上面这两个测试的问题都会在`$interpolate`的`while`循环。我们不再使用硬编码的`{{`和`}}`字符串，而使用`startSymbol`和`endSymbol`两个变量代替。另外，我们也不能假设这两个标识的字符串长度是2了，因此我们需要用变量的长度来计算切分字符串的位置：

_src/interpolate.js_

```js
while (index < text.length) {
  startIndex = text.indexOf(startSymbol, index);
  if (startIndex !== -1) {
    endIndex = text.indexOf(endSymbol, startIndex + startSymbol.length);
  }
  if (startIndex !== -1 && endIndex !== -1) {
    // if (startIndex !== index) {
    //   parts.push(unescapeText(text.substring(index, startIndex)));
    // }
    exp = text.substring(startIndex + startSymbol.length, endIndex);
    // expFn = $parse(exp);
    // expressions.push(exp);
    // expressionFns.push(expFn);
    // expressionPositions.push(parts.length);
    // parts.push(expFn);
    index = endIndex + endSymbol.length;
  } else {
    // parts.push(unescapeText(text.substring(index)));
    // break;
  }
}
```

这两个自定义的标识也应该要支持原油标识的反转义机制。如果开发者自定义开始标识为`FOO`，那模板中的`\F\O\O`字符串经过 interpolate 之后，就应该是`FOO`：

_test/interpolate_spec.js_

```js
it('supports unescaping for reconfigured symbols', function() {
  var injector = createInjector(['ng', function($interpolateProvider) {
    $interpolateProvider.startSymbol('FOO').endSymbol('OOF');
  }]);
  var $interpolate = injector.get('$interpolate');
  var interpFn = $interpolate('\\F\\O\\OmyExpr\\O\\O\\F');
  expect(interpFn({})).toEqual('FOOmyExprOOF');
});
```

这看上去更加棘手了。我们不能用那些专门用于反转义的、硬编码的正则表达式了。我们需要在运行时基于自定义的开始和结束标识组成新的正则表达式了。对于这两个标识，我们会改用[RegExp constructor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp)来创建正则表达式。我们会通过`escapeChar`函数（我们之后会讲到）对自定义的标识中的每一个字符进行处理，生成最终的正则表达式：

_src/interpolate.js_

```js
this.$get = ['$parse', function($parse) {
  var escapedStartMatcher =
    new RegExp(startSymbol.replace(/./g, escapeChar), 'g');
  var escapedEndMatcher =
    new RegExp(endSymbol.replace(/./g, escapeChar), 'g');

  // ...

}];
```

`escapeChar`函数会在字符之前加入三个反斜杠，而不是一、两个。我们需要在字符串中每一个反斜杠进行转义，所以最后就变成要六个反斜杠。

前两个反斜杠在正则表达式中会作为反斜杠符号本身的匹配器（`/\\/`）。而第三个反斜杠也是用于对原始字符进行转义。默认的开始和结束标识使用的花括号在正则表达式是有特殊含义，所以我们也需要对它们进行转义。

这样，最终生成出来的针对开始标识的正则表达式会是下面这样的：

```js
/\\\{\\\{/g
```

如果是我们在单元测试中自定义的开始标识，就是下面这样的：

```js
/\\\F\\\O\\\O/g
```

现在我们就可以对`unescapeText`函数进行修改，让它使用生成出来的正则表达式，而不是之前硬编码的表达式：

_src/interpolate.js_

```js
this.$get = ['$parse', function($parse) {
  // var escapedStartMatcher =
  //   new RegExp(startSymbol.replace(/./g, escapeChar), 'g');
  // var escapedEndMatcher =
  //   new RegExp(endSymbol.replace(/./g, escapeChar), 'g');

  function unescapeText(text) {
    return text.replace(escapedStartMatcher, startSymbol)
      .replace(escapedEndMatcher, endSymbol);
  }
  // ...
}];
```

上面的内容展示了我们怎么自定义表达式的开始和结束标识。但如果我们自定义了这两个标识，同时也使用了来自其他项目或开源库的第三方的代码会发生什么呢？会因为使用了跟第三方库不同的 interpolation 而导致程序异常吗？

答案是这种情况下不会使程序发生异常，我们依然可以正常使用第三方代码。那是因为在`$compile`中，我们会对所有指令模板进行“非统一化”，这意味着我们会把指令模板中所有的`{{`和`}}`标识换成我们自定义的。任何没有按照我们规范来的模板都会按照它们原来的处理方法进行处理。

下面就是一个例子：应用的开始和结束标志分别被自定义为`[[`和`]]`。它又使用了一个仍然使用`{{`和`}}`的指令。模板中的 interpolation 应该能够正常工作：

_test/compile_spec.js_

```js
it('denormalizes directive templates', function() {
  var injector = createInjector(['ng',
      function($interpolateProvider, $compileProvider) {
    $interpolateProvider.startSymbol('[[').endSymbol(']]');
    $compileProvider.directive('myDirective', function() {
      return {
        template: 'Value is {{myExpr}}'
      };
    }); 
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $rootScope.myExpr = 42;
    $compile(el)($rootScope);
    $rootScope.$apply();

    expect(el.html()).toEqual('Value is 42');
  });
});
```

你需要做的是必须通过一个函数`denormalizeTemplate`来处理所有的指令模板。在我们获取到这些模板时，我们就马上调用这个函数对模板进行处理。对于`applyDirectivesToNode`函数里出现的`template`属性来说，我们就这样调用：

_src/compile.js_

```js
if (directive.template) {
  // if (templateDirective) {
  //   throw 'Multiple directives asking for template';
  // }
  // templateDirective = directive;
  var template = _.isFunction(directive.template) ?
                    directive.template($compileNode, attrs) :
                    directive.template;
  template = denormalizeTemplate(template);
  $compileNode.html(template);
}
```

对于从`$http`获取到模板后调用的`compileTemplateUrl`中的`templateUrl`属性来说，就这样调用：

```js
function compileTemplateUrl(
    directives, $compileNode, attrs, previousCompileContext) {
    
  // ...
  
  $http.get(templateUrl).success(function(template) {
    template = denormalizeTemplate(template);
    
    // ...
  
  });

// ...

}
```

对于这个`denormalizeTemplate`函数，我们就定义在`$compile`服务中的`$get`方法中。

我们首先会从`$interpolate`服务中获取到当前的开始和结束标识，并与默认的标识进行对比。如果应用没有自定义表达式的标识，我们就把`denormalizeTemplate`函数定义为一个直接返回参数的函数，也就是对原内容不进行修改。如果应用用了自定义的标识，我们就通过正则表达式的替换功能把模板中的所有的默认标识都换成自定义的：

_src/compile.js_

```js
this.$get = ['$injector', '$parse', '$controller', '$rootScope',
  '$http', '$interpolate',
  function($injector, $parse, $controller, $rootScope, $http, $interpolate) {

    var startSymbol = $interpolate.startSymbol();
    var endSymbol = $interpolate.endSymbol();
    var denormalizeTemplate = (startSymbol === '{{' && endSymbol === '}}') ?
      _.identity :
      function(template) {
        return template.replace(/\{\{/g, startSymbol)
          .replace(/\}\}/g, endSymbol);
      };

    // ...
  
  }
];
```
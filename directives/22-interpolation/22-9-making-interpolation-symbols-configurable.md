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


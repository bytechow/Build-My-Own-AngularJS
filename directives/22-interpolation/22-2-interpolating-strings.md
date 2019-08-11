### 插入字符串（Interpolating Strings）

做好准备工作之后，我们就可以开始介绍`$interpolate`函数究竟会做些什么事了。

这个函数的基本工作是——接收一个字符串，这个字符串是被`{{ }}`包裹的，可能包含也可能不包含 Angular 表达式。然后会对字符串进行特定处理，最后返回一个函数。这个函数调用之后，会对字符串里面的所有表达式进行重新求值，并生成插值结果。这个函数需要一个上下文对象（通常是一个作用域），表达式将会在这个上下文语境内进行求值：

```js
var interpolateFn = $interpolate('{{a}} and {{b}}');
interpolateFn({a: 1, b: 2}); // => '1 and 2'
```

值得注意的是，这个效果很像`$parse`的作用：`$interpolate`和`parse`服务都会接收一个已经经过了一次“解析”的字符串。它们的返回值也是一个可以在指定作用域语境下进行求值的函数。这种相似性并不出奇，因为`$interpolate`服务实际上只是对`$parse`服务的一层封装：每个被花括号包裹的表达式都会单独使用`$parse`服务进行处理。

在我们讲述这种带有表达式的 interpolation 之前，我们先来看看最简单的一种 interpolation，也就是不带任何表达式的字符串。当传入一个简单的字符串，`$interpolate`应该会返回一个会产出相同字符串的函数：

_test/interpolate_spec.js_

```js
it('produces an identity function for static content', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  
  var interp = $interpolate('hello');
  expect(interp instanceof Function).toBe(true);
  expect(interp()).toEqual('hello');
});
```

要让这个单元测试通过，最简单的方法就是让函数直接返回传入的参数值：

_src/interpolate.js_

```js
this.$get = function() {

  function $interpolate(text) {
    return function interpolationFn() {
      return text;
    };
  }
  
  return $interpolate;
};
````

现在我们可以试着加入插值表达式，这能令事情变得更有趣一点。我们希望能够返回一个返回值是表达式求值结果的函数：

_test/interpolate_spec.js_

```js
it('evaluates a single expression', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  
  var interp = $interpolate('{{anAttr}}');
  expect(interp({
    anAttr: '42'
  })).toEqual('42');
});
```

在`$interpolate`函数里面，我们需要检查传入的字符串是否包含有`{{`和`}}`的标记。如果有的话，我们就把两个标记之间的字符串截取出来，并放到一个叫`exp`的变量中：

_src/interpolate.js_

```js
function $interpolate(text) {
  var startIndex = text.indexOf('{{');
  var endIndex = text.indexOf('}}');
  var exp;
  if (startIndex !== -1 && endIndex !== -1) {
    exp = text.substring(startIndex + 2, endIndex);
  }

  // return function interpolationFn() {
  //   return text;
  // };
  
}
```

我们会怎么处理这个子字符串呢？它是一个 Angular 表达式，所以我们应该对它进行解析。因为我们需要注入`$parse`服务，然后使用它来生成一个表达式函数：

_src/interpolate.js_

```js
this.$get = ['$parse', function($parse) {

  function $interpolate(text) {
  //   var startIndex = text.indexOf('{{');
  //   var endIndex = text.indexOf('}}');
    var exp, expFn;
    // if (startIndex !== -1 && endIndex !== -1) {
    //   exp = text.substring(startIndex + 2, endIndex);
      expFn = $parse(exp);
    // }

    // return function interpolationFn() {
    //   return text;
    // };

  }
  
  return $interpolate;
}];
```

在求值的时候，我们需要检查是否有表达式函数。如果有，我们需要在给定的上下文语境下对其进行求值运算。如果没有求值表达式的话，我们就直接返回字符串本身就可以了。

```js
return function interpolationFn(context) {
  if (expFn) {
    return expFn(context);
  } else {
    return text;
  }
};
```

现在虽然已经算是有一点小成果了。但目前的代码实现还是有太多的限制了，毕竟它只能处理只有一个表达式和静态文本两种情况。interpolation 也可以是包含多个表达式，这些表达式之间可能会有一些静态文本，我们也需要支持这种情况。

_test/interpolate_spec.js_

```js
it('evaluates many expressions', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  
  var interp = $interpolate('First {{anAttr}}, then {{anotherAttr}}!');
  expect(interp({
    anAttr: '42',
    anotherAttr: '43'
  })).toEqual('First 42, then 43!');
});
```

这意味着我们需要对文本进行遍历查找，并对所有出现的表达式进行收集。我们可以使用`while`循环语句，结束条件是遍历到字符串的最后一个字符。在每一次遍历中，我们会从当前的遍历索引开始，如果再找不到表达式的话，就结束循环：

_src/interpolate.js_

```js
function $interpolate(text) {
  var index = 0;
  var startIndex, endIndex, exp, expFn;
  while (index < text.length) {
    startIndex = text.indexOf('{{', index);
    endIndex = text.indexOf('}}', index);
    if (startIndex !== -1 && endIndex !== -1) {
    //   exp = text.substring(startIndex + 2, endIndex);
    //   expFn = $parse(exp);
      index = endIndex + 2;
    } else {
      break;
    }
  }

  // return function interpolationFn(context) {
  //   if (expFn) {
  //     return expFn(context);
  //   } else {
  //     return text;
  //   }
  // };
  
}
```

这个循环能找到所有的表达式，但还没能对它们进行收集。因此我们会新增一个数组来存放经过解析后的字符串的各个部分，这个字符串就命名为`parts`。在每次循环进行时，我们会把在表达式之前的静态文本和表达式函数放到这个数组中来。如果字符串中没有表达式，我们就把剩下的所有字符串都放到数组中去：

```js
function $interpolate(text) {
  // var index = 0;
  var parts = [];
  // var startIndex, endIndex, exp, expFn;
  while (index < text.length) {
    // startIndex = text.indexOf('{{', index);
    // endIndex = text.indexOf('}}', index);
    if (startIndex !== -1 && endIndex !== -1) {
      if (startIndex !== index) {
        parts.push(text.substring(index, startIndex));
      }
      // exp = text.substring(startIndex + 2, endIndex);
      // expFn = $parse(exp);
      parts.push(expFn);
      // index = endIndex + 2;
    } else {
      parts.push(text.substring(index));
      break;
    }
  }

  // return function interpolationFn(context) {
  //   if (expFn) {
  //     return expFn(context);
  //   } else {
  //     return text;
  //   }
  // };
  
}
```

这样我们就得到了一个保存 interpolation 各部分字符串的数组。它们中有些是静态文本，而有些是表达式函数。我们需要在求值阶段对这个数组进行遍历，以求出最终结果。

我们对这个数值进行 reduce 运算以求得一个字符串，在每次遍历中都会检查当前的元素是函数还是字符串。如果是函数就会以指定上下文求出结果进行拼接，如果是字符串，那就直接拼接上就好：

_src/interpolate.js_

```js
function $interpolate(text) {
  // var index = 0;
  // var parts = [];
  // var startIndex, endIndex, exp, expFn;
  // while (index < text.length) {
  //   startIndex = text.indexOf('{{', index);
  //   endIndex = text.indexOf('}}', index);
  //   if (startIndex !== -1 && endIndex !== -1) {
  //     if (startIndex !== index) {
  //       parts.push(text.substring(index, startIndex));
  //     }
  //     exp = text.substring(startIndex + 2, endIndex);
  //     expFn = $parse(exp);
  //     parts.push(expFn);
  //     index = endIndex + 2;
  //   } else {
  //     parts.push(text.substring(index));
  //     break;
  //   }
  // }

  return function interpolationFn(context) {
    return _.reduce(parts, function(result, part) {
      if (_.isFunction(part)) {
        return result + part(context);
      } else {
        return result + part;
      }
    }, '');
  };
  
}
```

这样我们就需要在`interpolate.js`中引入 LoDash：

```js
'use strict';

var _ = require('lodash');
```

这样，我们就有了一个简单的 interpolation 编译器！

但是还有一个问题。如果我们使用一个结束标志在前，开始标志在后的错误表达式，那结果就不是我们所希望的了：

_test/interpolate_spec.js_

```js
it('passes through ill-defned interpolations', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('why u no }}work{{');
  expect(interp({})).toEqual('why u no }}work{{');
});
```

我们可以把匹配规则改成是必须找到表达式的结束索引，这样保证一个表达式中的开始标志之后必须有结束标志。否则，我们就不把它们当作表达式进行求值：

```js
while (index < text.length) {
  // startIndex = text.indexOf('{{', index);
  if (startIndex !== -1) {
    endIndex = text.indexOf('}}', startIndex + 2);
  }
  // if (startIndex !== -1 && endIndex !== -1) {
  //   if (startIndex !== index) {
  //     parts.push(text.substring(index, startIndex));
  //   }
  //   exp = text.substring(startIndex + 2, endIndex);
  //   expFn = $parse(exp);
  //   parts.push(expFn);
  //   index = endIndex + 2;
  // } else {
  //   parts.push(text.substring(index));
  //   break;
  // }
}
```
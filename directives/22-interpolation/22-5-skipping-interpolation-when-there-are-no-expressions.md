### 若无表达式则跳过 interpolation（Skipping Interpolation When There Are No Expressions）

目前，`$interpolate`函数总是会返回一个函数，无论是有表达式还是没有表达式。这样的 API 是比较美观、统一的。

然而为了性能考虑，如果确实没有需要进行 interpolate 的，我们就不需要创建一个 interpolation 函数了。毕竟 interpolation 可能会在 watcher 中被经常调用，进行这个优化也有一定的道理。

要启用这个优化，我们可以传递一个布尔值`mustHaveExpressions`作为`$interpolate`服务的第二个参数。当这个参数被设置成`true`，那我们只会在发现字符串中有表达式才返回一个函数：

_test/interpolate_spec.js_

```js
it('does not return function when flagged and no expressions', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('static content only', true);
  expect(interp).toBeFalsy();
});
```

另外，即使设置了这个参数，但当字符串中含有表达式时，我们也会返回一个函数：

```js
it('returns function when flagged and has expressions', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('has an {{expr}}', true);
  expect(interp).not.toBeFalsy();
});
```

首先，`$interpolate`服务需要接受这个参数`mustHaveExpressions`。然后我们会在它内部加入一个局部变量，用于记录当前字符串中是否含有表达式。最终结果是，如果当前字符串有表达式，或没有指明一定要包含表达式，我们才会返回 interpolation 函数：

_src/interpolate.js_

```js
function $interpolate(text, mustHaveExpressions) {
  // var index = 0;
  // var parts = [];
  // var hasExpressions = false;
  // var startIndex, endIndex, exp, expFn;
  // while (index < text.length) {
  //   startIndex = text.indexOf('{{', index);
  //   if (startIndex !== -1) {
  //     endIndex = text.indexOf('}}', startIndex + 2);
  //   }
  //   if (startIndex !== -1 && endIndex !== -1) {
  //     if (startIndex !== index) {
  //       parts.push(unescapeText(text.substring(index, startIndex)));
  //     }
  //     exp = text.substring(startIndex + 2, endIndex);
  //     expFn = $parse(exp);
  //     parts.push(expFn);
      hasExpressions = true;
  //     index = endIndex + 2;
  //   } else {
  //     parts.push(unescapeText(text.substring(index)));
  //     break;
  //   }
  // }
  if (hasExpressions || !mustHaveExpressions) {
    // return function interpolationFn(context) {
    //   return _.reduce(parts, function(result, part) {
    //     if (_.isFunction(part)) {
    //       return result + stringify(part(context));
    //     } else {
    //       return result + part;
    //     }
    //   }, '');
    // };
  }
}
```

在本章后面我们会在`$interpolate`服务加入更多功能，但目前我们这样处理就可以了。
### 值的字符串化（Value Stringification）

interpolation 的结果一般都是字符串，但 Angular 表达式可以返回任何东西。也就是说，我们需要对不是字符串的值类型进行处理，让它们都能变成字符串的一部分。

比如，像`null`和`undefined`这一类值就会变成一个空字符串，而不是`"null"`或者`"undefined"`这种字符串：

_test/interpolate_spec.js_

```js
it('turns nulls into empty strings', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  
  var interp = $interpolate('{{aNull}}');
  expect(interp({
    aNull: null
  })).toEqual('');
});

it('turns undefneds into empty strings', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('{{anUndefned}}');
  expect(interp({})).toEqual('');
});
```

要处理这种情况，我们需要引入一个叫`stringify`的函数，它会接收一个值，并将值强制转换为一个字符串。我们会对每个表达式的值进行这个处理：

_src/interpolate.js_

```js
return function interpolationFn(context) {
  // return _.reduce(parts, function(result, part) {
  //   if (_.isFunction(part)) {
      return result + stringify(part(context));
  //   } else {
  //     return result + part;
  //   }
  // }, '');
};
```

正如我们之前说的，这个函数的第一个版本会把`null`和`undefined`这两个值转换为空字符串。其他的值都会直接使用本来字符拼接的方式进行强制转换：

```js
function stringify(value) {
  if (_.isNull(value) || _.isUndefned(value)) {
    return '';
  } else {
    return '' + value;
  }
}
```
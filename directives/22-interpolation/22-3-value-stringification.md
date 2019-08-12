### 值的字符串化（Value Stringification）

interpolation 的结果一般都是字符串，但 Angular 表达式可以返回任何东西。也就是说，我们需要对不是字符串的值类型进行处理，让它们都能变成字符串的一部分。

比如，像`null`和`undefined`这一类值就会变成一个空字符串，而不是`"null"`或者`"undefined"`这种字符串：

_test/interpolate\_spec.js_

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

数字和布尔值应该被强制转换为字符串：

_test/interpolate\_spec.js_

```js
it('turns numbers into strings', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('{{aNumber}}');
  expect(interp({aNumber: 42})).toEqual('42');
});
it('turns booleans into strings', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('{{aBoolean}}');
  expect(interp({aBoolean: true})).toEqual('true');
});
```

这些测试用例实际上已经可以通过了，这是因为我们直接把指串接到字符串中，这种情况下 JavaScript 原有的处理结果就跟我们预期的一样了。但是为了保证后面的开发过程不会影响这个预期效果，我们还是保留这两个测试。

对于复合值，也就是数组和对象，这项工作会更富有挑战性。如果要进行 interpolate 的是一个复合值，我们希望能够看到它的内容。而数组和对象在被强制转换为字符串的时候并不能提供比较有用的信息，因为我们需要把它们变成 JSON 字符串：

_test/interpolate\_spec.js_

```js
it('turns arrays into JSON strings', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

    var interp = $interpolate('{{anArray}}');
  expect(interp({anArray: [1, 2, [3]]})).toEqual('[1,2,[3]]');
});

it('turns objects into JSON strings', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

    var interp = $interpolate('{{anObject}}');
  expect(interp({anObject: {a: 1, b: '2'}})).toEqual('{"a":1,"b":"2"}');
});
```

这需要我们对这个值进行检查，如果是类型是`object`（对象和数组都属于对象类型）的话，我们就使用 `JSON.stringify`进行处理：

_src/interpolate.js_

```js
function stringify(value) {
  if (_.isNull(value) || _.isUndefined(value)) {
    return '';
  } else if (_.isObject(value)) {
    return JSON.stringify(value);
  } else {
    return '' + value;
  }
}
```

在生产环境中，我们很少需要把对象或数组渲染到 UI 上，但在开发时却有用武之地。你可以直接在 DOM 上打印数据结构，看看里面究竟有哪些值。


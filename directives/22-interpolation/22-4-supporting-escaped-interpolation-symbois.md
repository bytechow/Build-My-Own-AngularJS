### 支持加入逃逸 interpolation 的标识（Supporting Escaped Interpolation Symbols）

如果你想要在 UI 中不想把`{{`和`}}`这两个符号作为 interpolattion 的标识，而要直接输出它们，那我们就需要使用反斜杠符号来进行逃逸：`\{\{`或`\}\}`。在`$interpolate`服务中，如果遇到这种逃逸符号，我们就输出`{{`或`}}`就可以了。

> 实际上，Angular 官方文档推荐我们[要求从服务器返回的、用户提供的数据总是带有这种逃逸字符](https://docs.angularjs.org/api/ng/service/$interpolate#escaped-interpolation)。这实际上也是 Angular 对[OWASP 跨域脚本攻击预防指南](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet#RULE_.231_-_HTML_Escape_Before_Inserting_Untrusted_Data_into_HTML_Element_Content)的扩展。

下面这个测试用例就是对这一要求的检验。注意，如果我们需要在 interpolation 字符串中表示一个反斜杠，那在 JavaScript 代码中需要写上两个反斜杠：

_test/interpolate_spec.js_

```js
it('unescapes escaped sequences', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');

  var interp = $interpolate('\\{\\{expr\\}\\} {{expr}} \\{\\{expr\\}\\}');
  expect(interp({expr: 'value'})).toEqual('{{expr}} value {{expr}}');
});
```

每次我们在`parts`数组中插入静态文本，我们需要运行一个帮助函数`unescapeText`来进行转义字符的解析：

_src/interpolate.js_

```js
// while (index < text.length) {
//   startIndex = text.indexOf('{{', index);
//   if (startIndex !== -1) {
//     endIndex = text.indexOf('}}', startIndex + 2);
//   }
//   if (startIndex !== -1 && endIndex !== -1) {
//     if (startIndex !== index) {
      parts.push(unescapeText(text.substring(index, startIndex)));
  //   }
  //   exp = text.substring(startIndex + 2, endIndex);
  //   expFn = $parse(exp);
  //   parts.push(expFn);
  //   index = endIndex + 2;
  // } else {
    parts.push(unescapeText(text.substring(index)));
//     break; 
//   }
// }
```

在这个帮助函数中，我们可以使用两个正则表达式来将逃逸字符转变为实际要转换成的字符：

```js
function unescapeText(text) {
  return text.replace(/\\{\\{/g, '{{')
             .replace(/\\}\\}/g, '}}');
}
```
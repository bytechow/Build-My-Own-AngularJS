### 将表达式集成到作用域中去
#### Integrating Expressions to Scopes

以下 `Scope` 对外暴露的方法不仅可以支持原生的函数，同时还支持传入表达式字符串：

- `$watch`
- `$watchCollection`
- `$eval）`（同时还有包括相关的 `$apply` 和 `$evalAsync`）

在内部，`Scope` 会使用 `parse` 函数（也就是之后的 `$parse` 服务）把这些表达式都转换为函数。

由于 `Scope` 依然支持使用原生函数，我们需要检查一下传入的究竟是一个字符串，还是已经是一个函数。我们可以在 `parse` 函数来进行这个检查，如果我们发现要解析的是一个函数，就会把函数原封不动地返回回去：

_test/parse_spec.js_

```js
it('returns the function itself when given one', function() {
  var fn = function() {};
  expect(parse(fn)).toBe(fn);
});
```
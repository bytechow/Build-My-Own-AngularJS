### 处理布尔值属性（Handling Boolean Attributes）

HTML 属性中存在布尔值属性，比如文本框元素的`disabled`属性。布尔值属性很特别，他们不需要指定一个具体值。只要它们出现在元素中，它们的值就默认为“true”。

鉴于 JavaScript 有真值和假值的概念，如果我们能在布尔型属性支持真假值，这会大大方便我们。实际上，我们确实会对这一特性进行支持。Angular 会强制把所有布尔值属性放到属性对象中，并把这个属性值设置为`true`：

_test/compile_spec.js_

```js
it('sets the value of boolean attributes to true', function() {
  registerAndCompile(
    'myDirective',
    '<input my-directive disabled>',
    function(element, attrs) {
      expect(attrs.disabled).toBe(true);
    }
  );
});
```

但要特别注意，这样的强制指定并不是在元素的所有指令上都会出现，而只会出现在 HTML 标准中属性值为布尔值的属性上。除此以外的属性都不会进行这种特殊处理：

```js
it('does not set the value of custom boolean attributes to true', function() {
  registerAndCompile(
    'myDirective',
    '<input my-directive whatever>',
    function(element, attrs) {
      expect(attrs.whatever).toEqual('');
    }
  );
});
```

我们在`collectDirectives`中为属性对象设置属性值，所以我们也会在那里对布尔值属性使用属性值`true`:

_src/compile.js_

```js
attrs[normalizedAttrName] = attr.value.trim();
if (isBooleanAttribute(node, normalizedAttrName)) {
  attrs[normalizedAttrName] = true;
}
```

`isBooleanAttribute`这个新函数会进行两个检查：首先看属性是否 HTML 标准的布尔型属性，然后再看这个布尔型属性是否应用在对应的元素上：

```js
function isBooleanAttribute(node, attrName) {
  return BOOLEAN_ATTRS[attrName] && BOOLEAN_ELEMENTS[node.nodeName];
}
```

`BOOLEAN_ATTRS`常量中存放的是各种布尔型属性名称：

```js
var BOOLEAN_ATTRS = {
  multiple: true,
  selected: true,
  checked: true,
  disabled: true,
  readOnly: true,
  required: true,
  open: true,
}
```

`BOOLEAN_ELEMENTS`常量中存放的是我们需要关注的（可带有布尔型属性的）元素。由于使用 DOM 获取的节点名称都是大写，我们也会把常量对象里面的 key 写成大写：

```js
var BOOLEAN_ELEMENTS = {
  INPUT: true,
  SELECT: true,
  OPTION: true,
  TEXTAREA: true,
  BUTTON: true,
  FORM: true,
  DETAILS: true
};
```
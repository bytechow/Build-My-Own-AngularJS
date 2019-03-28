### 设置布尔型属性（Setting Boolean Properties）

当你设置一个属性时，Angular会对布尔型属性使用 jQuery 的 prop 方法注册属性：

_test/compile\_spec.js_

```js
it('sets prop for boolean attributes', function() {
  registerAndCompile(
    'myDirective',
    '<input my-directive>',
    function(element, attrs) {
      attrs.$set('disabled', true);
      expect(element.prop('disabled')).toBe(true);
    }
  );
});
`
```

另外，当我们选择不刷新 DOM 上的属性时，布尔型属性还是可以加入进去。当我们需要更新 DOM 属性（properties，比如`disabled`）不需要更新 DOM 或 HTML 特性时，这会变得很有用：

```js
it('sets prop for boolean attributes even when not fushing', function() {
  registerAndCompile(
    'myDirective',
    '<input my-directive>',
    function(element, attrs) {
      attrs.$set('disabled', true, false);
      expect(element.prop('disabled')).toBe(true);
    }
  );
});
```

在`$set`方法中，如果一个属性看起来像是布尔值，我们会使用 prop 方法来更新它，而无须理会`writeAttr`标志的值：

_src/compile.js_

```js
Attributes.prototype.$set = function(key, value, writeAttr) {
  // this[key] = value;

  if (isBooleanAttribute(this.$$element[0], key)) {
    this.$$element.prop(key, value);
  }

  // if (writeAttr !== false) {
  //   this.$$element.attr(key, value);
  // }
};
```

> 使用 prop 方法的其中一个原因是，并不是 jQuery 的所有版本都能一致地处理属性（prop）和特性（attr）。这个问题并不会在我们的项目中出现，因为我们已经限制了 jQuery 的版本，但如果要在 Angular 应用中使用 jQuery，你就需要意识到这个问题。




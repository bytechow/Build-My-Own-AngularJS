### 操作样式类（Manipulating Classes）

`Attributes`对象除了可以让属性变成可访问、可监视的，还提供了对样式类进行操作的帮助函数，可以用于对样式类的新增、删除和更新。但这些特性并不与样式类指令存在关联，它们仅会用于更新 DOM 元素上的 class 属性。

> `Attributes`的样式类操作还跟 Angular 的动画系统有紧密的联系，但本书不会包含这部分的内容。

你可以使用`Attributes`上的`$addClass`方法用于对元素新增样式类，使用`$removeClass`删除样式类：

_test/compile_spec.js_

```js
it('allows adding classes', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive></my-directive>',
    function(element, attrs) {
      attrs.$addClass('some-class');
      expect(element.hasClass('some-class')).toBe(true);
    }
  );
});

it('allows removing classes', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive class="some-class"></my-directive>',
    function(element, attrs) {
      attrs.$removeClass('some-class');
      expect(element.hasClass('some-class')).toBe(false);
    }
  );
});
```

当然，你可以使用 jQuery/jqLite 元素进行样式类的添加，其实这也就是我们在新方法会做的事情：

_src/compile.js_

```js
Attributes.prototype.$addClass = function(classVal) {
  this.$$element.addClass(classVal);
};

Attributes.prototype.$removeClass = function(classVal) {
  this.$$element.removeClass(classVal);
};
```

`Attributes`提供的第三个、也是最后一个关于样式类操作的方法比前面两个更有趣。它会接收两个参数：一个要更新的样式类集，另一个是旧的样式类集。它会对这两个样式集进行对比，然后添加第一个集中存在而在第二个不存在的样式类，并且删除在第二个样式集中存在，而在第一个集中不存在的样式类：

_test/compile_spec.js_

```js
it('allows updating classes', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive class="one three four"></my-directive>',
    function(element, attrs) {
      attrs.$updateClass('one two three', 'one three four');
      expect(element.hasClass('one')).toBe(true);
      expect(element.hasClass('two')).toBe(true);
      expect(element.hasClass('three')).toBe(true);
      expect(element.hasClass('four')).toBe(false);
    }
  );
});
```

所以这个新函数会接受新样式类集和旧样式类集作为参数，而这两个样式类集都用字符串表示。首先我们要做的是以空格为分隔符，把样式类集变成一个由单独样式类名组成的数组：

_src/compile.js_

```js
Attributes.prototype.$updateClass = function(newClassVal, oldClassVal) {
  var newClasses = newClassVal.split(/\s+/);
  var oldClasses = oldClassVal.split(/\s+/);
};
```

然后我们就使用正反向对比（利用LoDash）来筛选出要添加或删除的差异项，最后应用到对应的元素上即可：

```js
Attributes.prototype.$updateClass = function(newClassVal, oldClassVal) {
  // var newClasses = newClassVal.split(/\s+/);
  // var oldClasses = oldClassVal.split(/\s+/);
  var addedClasses = _.difference(newClasses, oldClasses);
  var removedClasses = _.difference(oldClasses, newClasses);
  if (addedClasses.length) {
    this.$addClass(addedClasses.join(' '));
  }
  if (removedClasses.length) {
    this.$removeClass(removedClasses.join(' '));
  }
};
```
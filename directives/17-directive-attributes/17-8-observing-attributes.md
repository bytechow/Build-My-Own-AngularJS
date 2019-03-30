### 监视属性（Observing Attributes）

就像我们之前讨论过那样，`Attributes`对象提供了在同一元素内不同指令间通信的一种方式。一个指令更改属性后，其他指令都可以访问到这个变化后的值。

如果我们能让一个指令对属性的变更时，另一个指令能够得到通知，这也会变得很有用。这样当属性变更时，我们就可以马上获知到变化。我们可以通过`$watch`对一个属性值进行监视，但 Angular 还提供了一种专门的机制来实现对属性的监视，也就是`$observe`：

_test/compile_spec.js_

```js
it('calls observer immediately when attribute is $set', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive some-attribute="42"></my-directive>',
    function(element, attrs) {

      var gotValue;
      attrs.$observe('someAttribute', function(value) {
        gotValue = value;
      });

      attrs.$set('someAttribute', '43');
      
      expect(gotValue).toEqual('43');
    }
  );
});
```

我们会在`Attributes`的原型上加入`$observe`方法中，这个方法会在调用`$set`方法后马上被调用，正如我们在测试用测中看到那样。

`Attibutes`对象会使用一个对象来存储监视器的注册函数，这个对象的 key 将会是需要监视的属性名称，而属性值会是一个监视器函数组成的数组：

_src/compile.js_

```js
Attributes.prototype.$observe = function(key, fn) {
  this.$$observers = this.$$observers || Object.create(null);
  this.$$observers[key] = this.$$observers[key] || [];
  this.$$observers[key].push(fn);
};
```

> 我们用`Object.create`方法和`null`参数来创建对象实例，而不是直接用对象字面量。这可以避免与原型上的属性产生冲突，比如`toString`和`constructor`。

在`$set`方法的最后，我们调用所有与该属性相关的监视器函数：

```js
Attributes.prototype.$set = function(key, value, writeAttr, attrName) {
  // this[key] = value;
  // if (isBooleanAttribute(this.$$element[0], key)) {
  //   this.$$element.prop(key, value);
  // }
  // if (!attrName) {
  //   if (this.$attr[key]) {
  //     attrName = this.$attr[key];
  //   } else {
  //     attrName = this.$attr[key] = _.kebabCase(key);
  //   }
  // } else {
  //   this.$attr[key] = attrName;
  // }
  // if (writeAttr !== false) {
  //   this.$$element.attr(attrName, value);
  // }
  if (this.$$observers) {
    _.forEach(this.$$observers[key], function(observer) {
      try {
        observer(value);
      } catch (e) {
        console.log(e);
      }
    });
  }
};
```
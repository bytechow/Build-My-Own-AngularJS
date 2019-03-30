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
  //https://www.mgtv.com/b/327378/5352386.html   } else {
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

注意我会把对监视器的调用放到一个`try...catch`代码块里面。我们在`$watches`和本书第一部分的事件监听器中也这样做了，原因是一样的：如果监视器抛出一个异常，我们也能保证其他监视器能被执行。

监视属性是对传统的监视器模式的应用。我们用`$watches`也能达到监听的目的，但使用`$observers`的优点是能能减少对`$digest`的压力。`$watch`函数会在每一次 digest 循环都会被执行，而`$observer`只会在设置属性时被调用，其余时间并不会占用任何 CPU 周期。

注意`$observers`并不会对在`Attributes`对象以外的属性变化作出响应。当你使用 jQuery 或是原生 DOM 的方式对某个元素的属性进行设置时，`$observers`并不会被触发。对属性的监视与`$set`方法紧密相关。这也是我们出于对性能的考虑而不使用`$watch`需要付出的代价。

所以我们知道，属性的`$observers`会在每次被`$set`时被调用，而监听器也会在初始注册后执行一次，这会在注册后的第一次`$digest`中发生：

test/compile_spec.js

```js
it('calls observer on next $digest after registration', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive some-attribute="42"></my-directive>',
    function(element, attrs, $rootScope) {

      var gotValue;
      attrs.$observe('someAttribute', function(value) {
        gotValue = value;
      });

      $rootScope.$digest();
      
      expect(gotValue).toEqual('42');
    }
  );
});
```

上面的测试单元的回调函数用到了第三个参数`$rootScope`。我们需要在`registerAndCompile`中传入：

```js
function registerAndCompile(dirName, domString, callback) {
  // var givenAttrs;
  // var injector = makeInjectorWithDirectives(dirName, function() {
  //   return {
  //     restrict: 'EACM',
  //     compile: function(element, attrs) {
  //       givenAttrs = attrs;
  //     }
  //   };
  // });
  injector.invoke(function($compile, $rootScope) {
    // var el = $(domString);
    // $compile(el);
    callback(el, givenAttrs, $rootScope);
  });
}
```

为了在`Attributes`中接入`$digest`，我们需要能访问到 Scope。我们可以在`CompileProvider`的`$get`方法中注入`$rootScope`：

_src/compile.js_

```js
this.$get = ['$injector', '$rootScope', function($injector, $rootScope) {
  // ...
}];
```

现在我们可以使用`$evalAsync`来在注册后的下一个`$digest`中执行回调。我们会直接用当前的属性值作为参数调用监听器函数：

```js
Attributes.prototype.$observe = function(key, fn) {
  var self = this;
  this.$$observers = this.$$observers || Object.create(null);
  this.$$observers[key] = this.$$observers[key] || [];
  this.$$observers[key].push(fn);
  $rootScope.$evalAsync(function() {
    fn(self[key]);
  });
};
```

即使监听器与 digest 并没有什么关系，但也会使用它来实现监听器第一次的调用。`Scope.$evalAsync`提供了一种方便的方式来完成这个操作。当然定时器也能够完成这个功能，但 Angular 就是这样处理的。

最后，就像`watcher`和事件监听器一样，observer 也能被移除：`$observe`方法的返回值就是一个可以让我们撤销 observer 的函数。

_test/compile_spec.js_

```js
it('lets observers be deregistered', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive some-attribute="42"></my-directive>',
    function(element, attrs) {
      
      var gotValue;
      var remove = attrs.$observe('someAttribute', function(value) {
        gotValue = value;
      });

      attrs.$set('someAttribute', '43');
      expect(gotValue).toEqual('43');
      
      remove();
      attrs.$set('someAttribute', '44');
      expect(gotValue).toEqual('43');
    }
  );
});
```

这个取消监听的函数跟之前我们写过的并没有差别：我们会在该属性上的所有监听器中找到对应回调函数的索引，并利用这个索引来删除这个监视器函数：

_src/compile.js_

```js
Attributes.prototype.$observe = function(key, fn) {
  var self = this;
  this.$$observers = this.$$observers || Object.create(null);
  this.$$observers[key] = this.$$observers[key] || [];
  this.$$observers[key].push(fn);
  $rootScope.$evalAsync(function() {
    fn(self[key]);
  });
  return function() {
    var index = self.$$observers[key].indexOf(fn);
    if (index >= 0) {
      self.$$observers[key].splice(index, 1);
    }
  };
};
```
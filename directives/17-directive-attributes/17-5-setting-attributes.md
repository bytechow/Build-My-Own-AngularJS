### 设置属性（Setting Attributes）

经过标准化后生成的属性对象是挺有用的，但如果我们能够通过这个属性对象来写入属性就更变得更强大了。为此，我们会通过属性对象的`$set`方法进行设置：

_test/compile\_spec.js_

```js
it('allows setting attributes', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive attr="true"></my-directive>',
    function(element, attrs) {
      attrs.$set('attr', 'false');
      expect(attrs.attr).toEqual('false');
    }
  );
});
```

属性对象的这个方法注册到原型上会更合适。这就需要我们会引入一个构造函数，Angular 就是这样做的：我们会在编译器的 provider 中注册一个名为`Attributes`的构造函数。它会接收一个元素节点作为它的参数：

_src/compile.js_

```js
this.$get = ['$injector', function($injector) {

  function Attributes(element) {
    this.$$element = element;
  }

  // ...
}];
```

在`compileNodes`函数中，我们现在会将之前属性对象从一个对象字面量，变成用这个构造函数来生成：

```js
function compileNodes($compileNodes) {
  // _.forEach($compileNodes, function(node) {
    var attrs = new Attributes($(node));
//     var directives = collectDirectives(node, attrs);
//     var terminal = applyDirectivesToNode(directives, node, attrs);
//     if (!terminal && node.childNodes && node.childNodes.length) {
//       compileNodes(node.childNodes);
//     }
//   });
}
```

现在我们需要用到的方法就是`$set`。要让第一个单元测试通过，我们只需要在该方法中把传入的属性设置到当前对象实例中即可：

```js
// function Attributes(element) {
//  this.$$element = element;
// }
Attributes.prototype.$set = function(key, value) {
  this[key] = value;
};
```

当你设置属性时，自然希望会把关联的 DOM 也一同更新了，而不仅仅是在 JavaScript 对象中变更：

_test/compile\_spec.js_

```js
it('sets attributes to DOM', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive attr="true"></my-directive>',
    function(element, attrs) {
      attrs.$set('attr', 'false');
      expect(element.attr('attr')).toEqual('false');
    }
  );
});
```

`$set`也确实这样处理，具体会通过 Attributes 构造函数在实例化时传入的元素：

_src/compile.js_

```js
Attributes.prototype.$set = function(key, value) {
 // this[key] = value;
  this.$$element.attr(key, value);
};
```

当然，我们也可以通过在调用`$set`时，把`false`第三个参数值，这样就可以禁用掉这个默认行为（必须为 false，而不能传入任何假值）：

_test/compile\_spec.js_

```js
it('does not set attributes to DOM when fag is false', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive attr="true"></my-directive>',
    function(element, attrs) {
      attrs.$set('attr', 'false', false);
      expect(element.attr('attr')).toEqual('true');
    }
  );
})
```

在实现代码中，我们会通过判断这个值是不是全等于`false`来决定是否要更新 DOM 上的属性：

_src/compile.js_

```js
Attributes.prototype.$set = function(key, value, writeAttr) {
//  this[key] = value;
  if (writeAttr !== false) {
 //   this.$$element.attr(key, value);
  }
};
```

但这个特性为什么是有用的呢？为什么你会希望对元素设置一个属性，而不想它的 DOM 上的属性也随之改变呢？是因为我们需要进行指令间通信，这与 DOM 操作无关，这也是为什么我们会使用`Attributes`构造函数生成的对象。

由于我们使用构造函数来构建属性对象，我们就可以在同一个元素的所有对象上共享同一个对象：

_test/compile\_spec.js_

```js
it('shares attributes between directives', function() {
  var attrs1, attrs2;
  var injector = makeInjectorWithDirectives({
    myDir: function() {
      return {
        compile: function(element, attrs) {
          attrs1 = attrs;
        }
      };
    },
    myOtherDir: function() {
      return {
        compile: function(element, attrs) {
          attrs2 = attrs;
        }
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-dir my-other-dir></div>');
    $compile(el);
    expect(attrs1).toBe(attrs2);
  });
})
```

正因为指令之间共享同一个属性对象，就可以利用这个属性对象来互相传递信息：

由于 DOM 访问比 JavaScript 访问要耗费更多时间和资源，Angular 为`$set`提供了可选的第三个参数用于优化性能。如果我们只希望让其他指令知晓属性变更，而不需要更改 DOM，这个参数将会很有用。


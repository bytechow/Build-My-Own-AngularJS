### 属性 interpolation（Attribute Interpolation）

第二种 interpolation 是在 DOM 属性上使用的。尽管它基本上跟文本节点上的 interpolation 十分类似，但由于属性可能会用于在元素的不同指令之间进行交互，这会比第一种复杂一点。我们需要保证开发这个功能后不影响指令之间的交互。

属性 interpolation 的基本行为跟文本节点 interpolation 是完全一样的。属性中的 interpolation 会在链接期间被替换，并会持续对变化进行检测：

_test/compile_spec.js_

```js
it('is done for attributes', function() {
  var injector = makeInjectorWithDirectives({});
  injector.invoke(function($compile, $rootScope) {
    var el = $('<img alt="{{myAltText}}">');
    $compile(el)($rootScope);

    $rootScope.$apply();
    expect(el.attr('alt')).toEqual('');
    
    $rootScope.myAltText = 'My favourite photo';
    $rootScope.$apply();
    expect(el.attr('alt')).toEqual('My favourite photo');
  });
});
```

开发过程跟文本节点的 interpolation 非常类似：我们会动态生成一个指令。我们通过函数`addAttrInterpolateDirective`来完成，我们会对 DOM 的所有元素的所有属性调用此函数。我们会给它传入`directives`集合，还有属性值和属性名：

_src/compile.js_

```js
function collectDirectives(node, attrs, maxPriority) {
  // var directives = [];
  // var match;
  // if (node.nodeType === Node.ELEMENT_NODE) {
  //   var normalizedNodeName = directiveNormalize(nodeName(node).toLowerCase());
  //   addDirective(directives, normalizedNodeName, 'E', maxPriority);
  //   _.forEach(node.attributes, function(attr) {
  
      // ...
  
      addAttrInterpolateDirective(directives, attr.value, normalizedAttrName);
  //     addDirective(directives, normalizedAttrName, 'A', maxPriority,
  //       attrStartName, attrEndName);
  
  //     // ...
  
  //   });
  
  //   // ...
  // }

  // ...
}
```

这个函数的基本结构是比较简单的：我们试图为属性值创建一个 interpolation 函数。如果找到了使用 interpolation 的属性，它会生成一个对 interpolation 函数返回值进行 watch 的指令，并在 listener 函数中设置属性值。

```js
function addAttrInterpolateDirective(directives, value, name) {
  var interpolateFn = $interpolate(value, true);
  if (interpolateFn) {
    directives.push({
      priority: 100,
      compile: function() {
        return function link(scope, element) {
          scope.$watch(interpolateFn, function(newValue) {
            element.attr(name, newValue);
          });
        };
      }
    });
  }
}
```

注意，这个指令的 priority 会设置为`100`，这能让这个指令先于大部分应用开发者自定义指令执行。

这就是属性 interpolation 的基本行为。现在我们可以开始讨论由属性 interpolation 所在元素上有其他指令的情况，这会带来了几个特殊的问题。

首先，其他属性可能正在通过`Attributes.$observe` observe 元素上的属性。原因可能是它们的独立作用域使用了`@`绑定的方法，也可能是它们显式地注册了 observer。当因 interpolation 出现了属性值的变更，我们希望这些 observer 都能被触发：

_test/compile_spec.js_

```js
it('fires observers on attribute expression changes', function() {
  var observerSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        link: function(scope, element, attrs) {
          attrs.$observe('alt', observerSpy);
        }
      }; 
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<img alt="{{myAltText}}" my-directive>');
    $compile(el)($rootScope);

    $rootScope.myAltText = 'My favourite photo';
    $rootScope.$apply();
    expect(observerSpy.calls.mostRecent().args[0])
      .toEqual('My favourite photo');
  });
});
```

测试未通过的原因是 observer 目前无法使用进行过 interpolate 的属性值，它使用的只是进行 interpolate 之前的原始文本。

我们可以通过改变设置属性的途径来解决这个问题。目前我们是通过 jQuery 的 attr 函数来进行设置的。实际上，我们应该使用 Attributes 对象的`$set`方法进行设置，因为它可以通知并调用 observer，同时也不耽误在 DOM 上设置属性：

_src/compile.js_

```js
return function link(scope, element, attrs) {
  scope.$watch(interpolateFn, function(newValue) {
    attrs.$set(name, newValue);
  });
};
```

当上一个单元测试失败时，我们能发现第二个问题。这个 observer 会在对`'{{myAltText}}'`进行 interpolation 之前进行调用。从应用开发者的视角来看，像这样获取值的方式并没有什么道理。我们希望 observer 只会在 interpolation 进行之后再被触发：

_test/compile_spec.js_

```js
it('fires observers just once upon registration', function() {
  var observerSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        link: function(scope, element, attrs) {
          attrs.$observe('alt', observerSpy);
        }
      }; 
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<img alt="{{myAltText}}" my-directive>');
    $compile(el)($rootScope);
    $rootScope.$apply();

    expect(observerSpy.calls.count()).toBe(1);
  });
});
```

在介绍 Attributes 的章节中，我们开发了一个功能可以让 observer 总可以在注册后的下一个 digest 中进行调用。现在，当我们需要对带有 interpolation 的属性进行 observe 时，希望能够跳过这一个过程。对这种属性的 observer 无论如何都会在下一个 digest 中进行调用，因为 interpolation 的 watcher 会对属性值进行设置。

如果属性是含有 interpolation 属性，我们会对这个属性的 observer 数组添加一个标识属性，名为`$$inter`。如果设置了设置了标识属性，我们就会跳过对 observer 的初始调用：

_src/compile.js_

```js
Attributes.prototype.$observe = function(key, fn) {
  // var self = this;
  // this.$$observers = this.$$observers || Object.create(null);
  // this.$$observers[key] = this.$$observers[key] || [];
  // this.$$observers[key].push(fn);
  // $rootScope.$evalAsync(function() {
    if (!self.$$observers[key].$$inter) {
      // fn(self[key]);
    }
  // });
  return function() {
    var index = self.$$observers[key].indexOf(fn);
    if (index >= 0) {
      self.$$observers[key].splice(index, 1);
    }
  };
};
```
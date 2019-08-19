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

当我们链接含有属性 interpolation 的指令时，我们可以设置这个标识。我们现在有 Attributes 对象，我们就可以把`$inter`标识加入到 Attributes 对象里面的 observer。但我们需要在这样做之前确保`$$observers`存在，毕竟这个数据结构是惰性加载的：

_src/compile.js_

```js
return function link(scope, element, attrs) {
  attrs.$$observers = attrs.$$observers || {};
  attrs.$$observers[name] = attrs.$$observers[name] || [];
  attrs.$$observers[name].$$inter = true;
  // scope.$watch(interpolateFn, function(newValue) {
  //     attrs.$set(name, newValue);
  // }); 
};
```

对于同一元素上的其他指令来说，我们还有一个问题要解决，就是我们需要在这些指令进行链接前处理好 interpolation。通常，当要访问一个指令上的属性时，应用开发者并不一定非得考虑它们是否经过了 interpolate。这是 Angular 框架需要解决的问题。目前，我们还没有很好地解决这个问题：

_test/compile_spec.js_

```js
it('is done for attributes by the time other directive is linked', function() {
  var gotMyAttr;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        link: function(scope, element, attrs) {
          gotMyAttr = attrs.myAttr;
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="{{myExpr}}"></div>');
    $rootScope.myExpr = 'Hello';
    $compile(el)($rootScope);

    expect(gotMyAttr).toEqual('Hello');
  });
});
```

在这个单元测试中，我们希望访问`myDirective`上的属性值时结果为`Hello`，但目前的结果是`{{myExpr}}`。我们需要修正这个问题。

造成这个问题的原因有几个。原因之一是 interpolation 是会在链接后的第一个 digest 循环中进行，因为我们是在 watcher 的 listener 函数中进行 interpolation 过程的。实际上，我们应该在设置 watcher 之前就要进行一次 interpolation，这样，我们就能在（链接后的）任何一个 digest 之前已经获得经过 interpolate 的内容。

_src/compile.js_

```js
compile: function() {
  return function link(scope, element, attrs) {
    // attrs.$$observers = attrs.$$observers || {};
    // attrs.$$observers[name] = attrs.$$observers[name] || [];
    // attrs.$$observers[name].$$inter = true;
    attrs[name] = interpolateFn(scope);
    // scope.$watch(interpolateFn, function(newValue) {
    //   attrs.$set(name, newValue);
    // });
  };
}
```

但我们的单元测试仅凭这个处理还是没法通过。造成测试未通过的第二个原因出现在指令的应用顺序上。含有属性 interpolation 的指令的 priority 是`100`，这意味着它会在采用默认优先级的“普通”指令之前进行编译。然而，我们是在指令的 post-link 函数设置 interpolation 的，如果你回忆起跟 post-link 有关的章节，post-link 函数是以与注册顺序相反的顺序执行的。这就导致“普通”指令是在含 interpolation 的指令之前执行的。

要是我们换成_pre-link函数_，那么事情就会像我们预想的那样进行：

```js
compile: function() {
  return {
    pre: function link(scope, element, attrs) {
      // attrs.$$observers = attrs.$$observers || {};
      // attrs.$$observers[name] = attrs.$$observers[name] || [];
      // attrs.$$observers[name].$$inter = true;
      // attrs[name] = interpolateFn(scope);
      // scope.$watch(interpolateFn, function(newValue) {
      //   attrs.$set(name, newValue);
      // });
    }
  };
}
```

现在，关于属性的独立作用域绑定问题又该如何呢？前面我们已经介绍过`@`属性绑定是借助 observer 实现的，而我们已经考虑过 observer 了吗？（？）

一般来说，bindings 会起作用，但关于 bindings 的初始值还存在一个问题。在一个独立作用域指令进行链接时，它的属性 bindings 现在指向的是在 interpolate 之前的属性值，这跟我们之前在普通属性中遇到的问题非常类似：

_test/compile_spec.js_

```js
it('is done for attributes by the time bound to iso scope', function() {
  var gotMyAttr;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {myAttr: '@'},
        link: function(scope, element, attrs) {
          gotMyAttr = scope.myAttr;
        }
      }; 
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="{{myExpr}}"></div>');
    $rootScope.myExpr = 'Hello';
    $compile(el)($rootScope);

    expect(gotMyAttr).toEqual('Hello');
  });
});
```

问题出自对属性进行的 isolate binding 的设置过程。我们是在`initializeDirectiveBindings`中设置 binding 指向的初始值的。这个函数是在任何指令进行链接之前调用的，包括含有属性 interpolation 的指令：

_src/compile.js_

```js
case '@':
  attrs.$observe(attrName, function(newAttrValue) {
    destination[scopeName] = newAttrValue;
  });
  if (attrs[attrName]) {
    destination[scopeName] = attrs[attrName];
  }
  break;
```

如果我们在设置初始值的时候就进行一次 interpolation，就可以解决这个问题了：

```js
case '@':
  attrs.$observe(attrName, function(newAttrValue) {
    destination[scopeName] = newAttrValue;
  });
  if (attrs[attrName]) {
    destination[scopeName] = $interpolate(attrs[attrName])(scope);
  } 
  break;
```

这里我们使用了`$interpolate`，但并没有传入`mustHaveExpressions`标识，这样就可以调用 interpolation 函数而不管到底字符串里面是否有 interpolation。

带有指令的元素上还会发生另一件事，就是指令会在编译期间操作元素属性。下面这个单元测试演示了这样一种情况：一个指令会在它的`compile`函数中对一个属性的值进行更改。问题是目前属性的 interpolation 函数并未能兼顾这种情况，仍是在值进行替换之前就使用了属性值，这是我们不希望发生的情况：

_test/compile_spec.js_

```js
it('is done for attributes so that compile-time changes apply', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        compile: function(element, attrs) {
          attrs.$set('myAttr', '{{myDifferentExpr}}');
        }
      }; 
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="{{myExpr}}"></div>');
    $rootScope.myExpr = 'Hello';
    $rootScope.myDifferentExpr = 'Other Hello';
    $compile(el)($rootScope);
    $rootScope.$apply();

    expect(el.attr('my-attr')).toEqual('Other Hello');
  });
});
```

这里的问题在于属性 interpolation 函数是在编译期间生成的，而指令替换属性值发生在编译后期。(?)

我们需要对带属性 interpolation 的指令链接函数增加一个检验语句，如果属性值在编译期间发生了改变，就再次生成 interpolation 函数：

_src/compile.js_

```js
pre: function link(scope, element, attrs) {
  var newValue = attrs[name];
  if (newValue !== value) {
    interpolateFn = $interpolate(newValue, true);
  }

  // attrs.$$observers = attrs.$$observers || {};
  // attrs.$$observers[name] = attrs.$$observers[name] || [];
  // attrs.$$observers[name].$$inter = true;

  // attrs[name] = interpolateFn(scope);
  // scope.$watch(interpolateFn, function(newValue) {
  //   attrs.$set(name, newValue);
  // });
}
```

在编译期间，指令也可能会完全移除某个属性。如果这个属性是有 interpolation 正在执行，我们不应该让它进行执行了：

_test/compile_spec.js_

```js
it('is done for attributes so that compile-time removals apply', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        compile: function(element, attrs) {
          attrs.$set('myAttr', null);
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="{{myExpr}}"></div>');
    $rootScope.myExpr = 'Hello';
    $compile(el)($rootScope);
    $rootScope.$apply();
    
    expect(el.attr('my-attr')).toBeFalsy();
  });
});
```

目前在这个测试中，不仅仅是 interpolation 无法正常运作，目前会在链接阶段就抛出一个一场。我们不希望这种情况发生。

我们需要再加上一层检验，如果属性没有没有更新值，我们就不需要重新生成 intepolation 函数。而且，在这种情况下，我们也要提前退出当前链接函数的执行，以免再设置多余的 watcher：

_src/compile.js_

```js
pre: function link(scope, element, attrs) {
  // var newValue = attrs[name];
  if (newValue !== value) {
    interpolateFn = newValue && $interpolate(newValue, true);
  }
  if (!interpolateFn) {
    return;
  }

  // attrs.$$observers = attrs.$$observers || {};
  // attrs.$$observers[name] = attrs.$$observers[name] || [];
  // attrs.$$observers[name].$$inter = true;
  
  // attrs[name] = interpolateFn(scope);
  // scope.$watch(interpolateFn, function(newValue) {
  //   attrs.$set(name, newValue);
  // });
}
```

最后，我们需要加入一个预防措施来禁止开发者在诸如`onclick`之类的事件处理函数使用 interpolation。即使开发者这样做了，interpolation 也不会生效，因为原生的事件处理器是在 Angular digest 循环之外触发的，它们也没有访问任何 scope 的权限。由于这个注意事项并没有对开发者进行明确说明，尤其是对框架的初学者。如果你尝试这样做，Angular 会进行禁止并抛出一个异常。如果真的要在事件处理器上使用，我们应该使用`ng-` 开头的事件处理器指令。

_test/compile_spec.js_

```js
it('cannot be done for event handler attributes', function() {
  var injector = makeInjectorWithDirectives({});
  injector.invoke(function($compile, $rootScope) {
    $rootScope.myFunction = function() { };
    var el = $('<button onclick="{{myFunction()}}"></button>');
    expect(function() {
      $compile(el)($rootScope);
    }).toThrow();
  }); 
});
```

我们在属性 interpolation 指令的`pre-link`函数之前会加上一个检查，看属性名称是否以`on`开头或就是`formaction`。这就能涵盖目前，乃至于以后的事件属性标准：

_src/compile.js_

```js
pre: function link(scope, element, attrs) {
  if (/^(on[a-z]+|formaction)$/.test(name)) {
    throw 'Interpolations for HTML DOM event attributes not allowed';
  }
  // ...
}
```
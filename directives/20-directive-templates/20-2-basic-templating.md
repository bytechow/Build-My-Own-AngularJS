### 基本模板（Basic Templating）

指令模板背后的基本思想也不难。一个指令可以在定义中指定一个`template`属性。这个属性会放入一个 HTML 字符串。当指令被编译到一个 DOM 元素上时，会根据 HTML 字符串生成要插入到元素中的内容。

这里是一个实例：

_test/compile_spec.js_

```js
describe('template', function() {
  it('populates an element during compilation', function() {
    var injector = makeInjectorWithDirectives('myDirective', function() {
      return {
        template: '<div class="from-template"></div>'
      };
    });
    injector.invoke(function($compile) {
      var el = $('<div my-directive></div>');
      $compile(el);
      expect(el.fnd('> .from-template').length).toBe(1);
    });
  });
});
```

如果使用指令模板的元素已经有内容了，这些内容将会被指令模板取代。在指令应用上时，之前指令元素的子元素就不再存在了：

```js
it('replaces any existing children', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      template: '<div class="from-template"></div>'
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive><div class="existing"></div></div>');
    $compile(el);
    expect(el.fnd('> .existing').length).toBe(0);
  });
});
```

模板内容并不只能是静态 HTML。模板内容一样会进行编译，这样里面的指令也一样会被应用上。我们可以通过在指令模板中应用一个带指令的元素进行测试，看这个元素的 compile 函数有没有被调用：

```js
it('compiles template contents also', function() {
  var compileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        template: '<div my-other-directive></div>'
      };
    },
    myOtherDirective: function() {
      return {
        compile: compileSpy
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive></div>');
    $compile(el);
    expect(compileSpy).toHaveBeenCalled();
  });
});
```

以上就是指令模板的基本行为了。下面我们来看看怎么实现。

指令模板是在编译过程中应用上的，具体是在`applyDirectivesToNode`函数中。在这个函数中，当我们对每个指令进行遍历时，我们可以检查它们是否有用到模板。如果有用到模板，我们就可以用模板内容来取代元素原先的内容：

_src/compile.js_

```js
s
_.forEach(directives, function(directive) {
  if (directive.$$start) {
    $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  }
  if (directive.priority < terminalPriority) {
    return false;
  }
  if (directive.scope) {
    if (_.isObject(directive.scope)) {
      if (newIsolateScopeDirective || newScopeDirective) {
        throw 'Multiple directives asking for new/inherited scope';
      }
      newIsolateScopeDirective = directive;
    } else {
      if (newIsolateScopeDirective) {
        throw 'Multiple directives asking for new/inherited scope';
      }
      newScopeDirective = newScopeDirective || directive;
    }
  }
  if (directive.compile) {
    var linkFn = directive.compile($compileNode, attrs);
    var isolateScope = (directive === newIsolateScopeDirective);
    var attrStart = directive.$$start;
    var attrEnd = directive.$$end;
    var require = directive.require;
    if (_.isFunction(linkFn)) {
      addLinkFns(null, linkFn, attrStart, attrEnd, isolateScope, require);
    } else if (linkFn) {
      addLinkFns(linkFn.pre, linkFn.post, attrStart, attrEnd, isolateScope,
        require);
    }
  }
  if (directive.controller) {
    controllerDirectives = controllerDirectives || {};
    controllerDirectives[directive.name] = directive;
  }
  if (directive.template) {
    $compileNode.html(directive.template);
  }
  if (directive.terminal) {
    terminal = true;
    terminalPriority = directive.priority;
  }
});
```
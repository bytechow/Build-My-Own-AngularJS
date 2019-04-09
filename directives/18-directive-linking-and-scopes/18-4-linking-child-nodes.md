### 链接子节点（Linking Child Nodes）

目前我们只实现了对单层的 DOM 结构进行了链接。实际上我们需要对编译后的整个 DOM 树进行链接，也就是包括 DOM 中的所有后代节点，下面我们会对此进行支持。

当我们对在不同层级上都有指令的 DOM 树进行链接时，我们会先从低层级的指令进行链接。

_test/compile_spec.js_

```js
it('links directive on child elements first', function() {
  var givenElements = [];
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      link: function(scope, element, attrs) {
        givenElements.push(element);
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-directive></div></div>');
    $compile(el)($rootScope);
    expect(givenElements.length).toBe(2);
    expect(givenElements[0][0]).toBe(el[0].firstChild);
    expect(givenElements[1][0]).toBe(el[0]);
  });
});
```

在这个测试中，我们会先对使用到`myDirective`的元素进行收集。我们会对两个元素应用该指令：一个元素节点和它的父节点。然后我们希望子元素是第一个被收集到的元素。

在编译过程中，我们目前是在`compileNodes`方法中通过递归访问节点的`childNodes`属性来对子节点进行处理。由于`compileNodes`目前返回的是一个复合链接函数，我们需要把递归调用返回的链接函数也保存起来，以便之后对子节点进行链接。因此我们需要在对每个节点进行编译时收集最多两个链接函数：该节点的链接函数和其子节点的复合链接函数。只要这两个链接函数存在任意一个，我们都会把它放到`linkFns`数组里面：

_src/compile.js_

```js
function compileNodes($compileNodes) {
  // var linkFns = [];
  // _.forEach($compileNodes, function(node, idx) {
  //   var attrs = new Attributes($(node));
  //   var directives = collectDirectives(node, attrs);
  //   var nodeLinkFn;
  //   if (directives.length) {
  //     nodeLinkFn = applyDirectivesToNode(directives, node, attrs);
  //   }
    var childLinkFn;
    // if ((!nodeLinkFn || !nodeLinkFn.terminal) &&
    //   node.childNodes && node.childNodes.length) {
      childLinkFn = compileNodes(node.childNodes);
    // }
    if (nodeLinkFn || childLinkFn) {
      // linkFns.push({
      //   nodeLinkFn: nodeLinkFn,
        childLinkFn: childLinkFn,
  //       idx: i
  //     });
    }
  // });

  // ...

}
```

现在，当在复合链接函数调用节点链接函数时，我们会新增一个参数：子节点的链接函数。我们希望节点链接函数在调用时也可以对其子节点的链接函数进行处理.

_src/compile.js_

```js
function compositeLinkFn(scope, linkNodes) {
  _.forEach(linkFns, function(linkFn) {
    linkFn.nodeLinkFn(
      linkFn.childLinkFn,
      scope,
      linkNodes[linkFn.idx]
    );
  });
}
```

子节点的链接函数事实上成为了节点链接函数的第一个参数，这会影响一些已经存在的单元测试。我们会通过更新`nodeLinkFn`方法进行修复。具体来说，我们会让传入的子节点链接函数先于节点的链接函数执行：

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  if (childLinkFn) {
    childLinkFn(scope, linkNode.childNodes);
  }
  // _.forEach(linkFns, function(linkFn) {
  //   var $element = $(linkNode);
  //   linkFn(scope, $element, attrs);
  // });
}
```

加上这个处理后，之前的单元测试就能正常通过了，而且我们也确实可以对子节点进行链接了。当一个节点被链接，它的子节点也会进行链接。

这种实现方法的问题在于，如果一个节点并没有使用任何指令，它就不会被链接，相应地，它的子节点也不会被链接，即使子节点上是有指令的。举例来说，像下面这个测试用例就会失败了：

_test/compile_spec.js_

```js
it('links children when parent has no directives', function() {
  var givenElements = [];
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      link: function(scope, element, attrs) {
        givenElements.push(element);
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div><div my-directive></div></div>');
    $compile(el)($rootScope);
    expect(givenElements.length).toBe(1);
    expect(givenElements[0][0]).toBe(el[0].frstChild);
  });
});
```

在这个测试中，我们希望带有指令的子元素会被链接，但事实证明是没有的。事实上，这个测试会因为我们尝试调用复合链接函数中的一个不存在的节点链接函数而抛出错误。

要解决这个问题，我们可以新增一个检查，检查传入的节点链接函数是否存在。如果有，我们按照之前的逻辑进行处理——调用它并希望它继续去链接子元素。但如果没有，我们会从复合函数中直接调用子节点的链接函数。

_src/compile.js_

```js
function compositeLinkFn(scope, linkNodes) {
  // _.forEach(linkFns, function(linkFn) {
    if (linkFn.nodeLinkFn) {
      // linkFn.nodeLinkFn(
      //   linkFn.childLinkFn,
      //   scope,
      //   linkNodes[linkFn.idx]
      // );
    } else {
      linkFn.childLinkFn(
        scope,
        linkNodes[linkFn.idx].childNodes
      );
    }
  // });
}
```

要记住，`childLinkFn`是子节点的复合链接函数，所以需要接收两个参数：作用域对象和要链接的节点。
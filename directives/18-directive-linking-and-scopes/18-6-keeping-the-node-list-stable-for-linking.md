### 在链接时保持节点列表稳定（Keeping The Node List Stable for Linking）

正如我们之前提及的，目前我们设置复合链接函数需要把编译节点和链接节点进行一对一的绑定，但当对 DOM 结构进行改变时，我们就不能保证这个联系的稳定性了。因为 DOM 操作通常就是指令要干的事，那这就可能产生问题了。举例来说，如果我们有一个要对目标元素插入兄弟元素，而这个元素正处于链接过程，这就会打乱我们的链接过程：

_test/compile\_spec.js_

```js
it('stabilizes node list during linking', function() {
  var givenElements = [];
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      link: function(scope, element, attrs) {
        givenElements.push(element[0]);
        element.after('<div></div>');
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div><div my-directive></div><div my-directive></div></div>');
    var el1 = el[0].childNodes[0],
      el2 = el[0].childNodes[1];
    $compile(el)($rootScope);
    expect(givenElements.length).toBe(2);
    expect(givenElements[0]).toBe(el1);
    expect(givenElements[1]).toBe(el2);
  });
});
```

在这个单元测试中，我们会创建一个在元素后拆入新元素的指令。我们会对两个兄弟元素应用这个指令，并希望两个都会被链接。但事实是其中一个插入的元素会进行链接，而且指令第二次应用时，要编译的元素已经不同于要用于链接的元素了。这就不是我们想要的。

我们可以通过在可以节点链接函数之前保留一份节点列表的拷贝来解决这个问题。不同于原生的节点列表，这个列表在链接期间不允许进行元素的各种变换操作。我们可以通过遍历记录`linkFns`数组中的索引：

_src/compile.js_

```js
function compositeLinkFn(scope, linkNodes) {
  var stableNodeList = [];
  _.forEach(linkFns, function(linkFn) {
    var nodeIdx = linkFn.idx;
    stableNodeList[nodeIdx] = linkNodes[nodeIdx];
  });

  // _.forEach(linkFns, function(linkFn) {
  //   if (linkFn.nodeLinkFn) {
  //     linkFn.nodeLinkFn(
  //       linkFn.childLinkFn,
  //       scope,
        stableNodeList[linkFn.idx]
    //   );
    // } else {
    //   linkFn.childLinkFn(
    //     scope,
        stableNodeList[linkFn.idx].childNodes
  //     );
  //   }
  // });
}
```




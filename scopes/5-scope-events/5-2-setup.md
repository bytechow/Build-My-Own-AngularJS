### 开发准备
#### Setup

我们还是会在 scope 对象中添加这个功能，所以代码还是会放到 `src/scope.js` 和 `test/scope_spec.js` 两个文件里。首先，我们会在测试文件最顶层的 `describe` 代码块中加入一个新的 `describe` 代码块：

_test/scope_spec.js_

```js
describe('Events', function() {

  var parent;
  var scope;
  var child;
  var isolatedChild;

  beforeEach(function() {
    parent = new Scope();
    scope = parent.$new();
    child = scope.$new();
    isolatedChild = scope.$new(true);
  });
});
```

我们要开发的功能与作用域继承树紧密相关，所以我们会在 `beforeEach` 函数中先创建一个作用域继承树。在这个树中，有一个父作用域和两个子作用域，其中一个子作用域是孤立的（isolated）。这就可以涵盖我们需要测试的关于继承的所有内容。
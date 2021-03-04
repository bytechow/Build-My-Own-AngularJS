### 开发准备
#### Setup

我们依然是在 Scope 对象上开发这个功能，因此代码还是会放到 `src/scope.js` 和 `test/scope_spec.js` 两个文件里。首先，我们会在测试文件最顶层的 `describe` 代码块中新增一个内嵌的 `describe` 代码块：

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

我们要开发的功能与作用域继承树紧密相关，所以我们会在 `beforeEach` 函数中预先创建一个作用域继承树。作用域继承树里面有一个父作用域和两个子作用域，其中一个子作用域是隔离的（isolated），这就足以测试与继承相关的内容了。
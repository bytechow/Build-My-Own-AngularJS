### 只实例化一次全局对象（Initializing The Global Just Once）

angular 全局对象存储已注册的模块，其本质上也是全局状态的储存器。这意味着我们需要找到管理状态的方法。首先，我们希望单元测试之间互不干扰，这需要在每个单元测试开始之前删除已存在的 angular 全局对象：

```js
beforeEach(function(){
  delete window.angular;
})
```

另外，在 setupModuleLoader 函数中，我们需要保证 angular 全局对象不会被覆盖，即使我们调用了多次 setupModuleLoader 函数。在测试中，当我们调用 setupModuleLoader 两次时，第一次调用后和第二次调用后访问 angular 全局对象，其结果应该指向同一个对象：

```js
it('creates angular just once', function() {
  setupModuleLoader(window);
  var ng = window.angular;
  setupModuleLoader(window);
  expect(window.angular).toBe(ng);
});
```

我们可以通过简单的检测来解决这个问题：

```js
function setupModuleLoader(window) {
  var angular = (window.angular = window.angular || {});
}
```

之后很快我们就再一次使用这种“单例模式”，所以我们先抽象出一个通用的函数 ensure，这个函数将会接收三个参数：一个对象 obj，对象的属性名 name 和一个“”工厂函数“ factory。当对象 obj.name 不存在，才会调用工厂函数产生一个值，并赋值到obj.name：

```js
function setupModuleLoader(window) {
  var ensure = function(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  };
  var angular = ensure(window, 'angular', Object);
}
```

这里我们先用 Object 构造函数生成一个空对象，赋值给 angular 全局对象。注意Object\( \) 与 new Object\( \) 在效果上一样的。


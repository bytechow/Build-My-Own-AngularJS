### 安装 Lo-Dash 和 jQuery（Include Lo-Dash And jQuery）

官方的 Angular 项目本身并不依赖任何第三方库（但如果项目引入了 jQuery 库，Angular 会直接使用这个库）。在我们的开发过程中，为了集中精力研究 Angular 框架的原理，我们会把一些底层细节交给现成的第三方库处理。

我们会找第三方库来帮忙处理两类底层操作：

- 数组和对象的操作，比如遍历，相等性比较和克隆，我们会交给 [Lo-Dash](https://lodash.com/) 来完成。
- DOM 查询与操作，我们会交给 [jQuery](https://jquery.com/) 来完成。

这两个库都可以通过 NPM 进行安装：

```bash
npm install --save lodash@4 jquery
``` 

安装方法跟之前安装 Browserify 和 Karma 时一模一样，唯一不同的是这次使用的参数是 `--save`，不是`--save-dev`。这个参数意味着，我们不仅在开发阶段会用到这两个库，在程序运行时也依赖它们。

要两个库是否可以正常使用，我们可以对 “Hello, world!” 程序做一些改动。安装 LoDash 的 NPM 包后，我们可以直接 `require` 这个库：

_src/hello.js_

```js
var _ = require('lodash');

module.exports = function sayHello(to){
  return _.template('Hello, <%= name %>!')({name: to})
}
```

我们将要打招呼的对象名称作为参数传入，然后利用 Lo-Dash 的 [template](https://lodash.com/docs#template) 函数生成结果字符串。

相应地，单元测试也需要更新一下：

_test/hello_spec.js_

```js
var sayHello = require('../src/hello');

describe('Hello', function() {
  
  it('says hello', function() {
    expect(sayHello('Jane')).toBe('Hello, Jane!');
  });

});
```
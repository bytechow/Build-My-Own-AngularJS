## 安装 Lo-Dash 和 jQuery（Include Lo-Dash And jQuery）

Angular 本身并不依赖任何第三方库（虽然如果引入了 jQuery，它也确实会用）。但在我们的开发过程中，为了集中精力研究 Angular 框架的原理，我们可以将一些底层细节交给已有的第三方库来处理。

有两类底层操作我们可以找第三方库来帮忙：

- 数组和对象的操作，比如遍历，相等性比较和克隆，我们会交给 Lo-Dash 来完成。
- DOM 查询与操作，我们会交给 jQuery 来完成。

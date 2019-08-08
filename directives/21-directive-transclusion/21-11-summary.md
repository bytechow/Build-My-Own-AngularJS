### 本章总结（Summary）

transclusion 是一个比较大的主题，既跟代码开发的复杂性有关，也跟它需要融入到`$compile`服务有关。为了实现这个功能，我们已经接触了`compile.js`很多不同的部分。

我们在实现这个功能的过程中，实际上对 DOM 编译器加入了两个重要的功能：一个是把模板中一部分内容转移到另一个模板（普通的 transclusion），另一个是控制一个给定元素会在什么时间和如何多次进行链接，并添加到 DOM 上（full element transclusion）。
### 一种全新的语言
#### A Whole New Language

那到底什么是 Angular 表达式呢？它们不就是插入到 HTML 标签之中的 JavaScript 代码片段吗？（这种说法）很接近了，但还不够准确。

Angular 表达式是为访问和操作作用域对象上的数据而定制的，除此之外别无它用。显而易见的是，表达式非常接近 JavaScript，正如 Miško Hevery 指出的，你可以只用几行 JavaScript 代码就可以实现表达式系统的大部分功能：

```js
function parse(expr){
  return function(scope){
    with (scope) {
      return eval(expr);
    }
  }
}
````

这个函数会接收一个表达式字符串，然后返回一个函数。这个函数会把这个字符串解析为 JavaScript 代码然后执行。同时，函数里使用了 `with` 语法把这段代码的上下文设定为一个特定的作用域对象。

但这种实现方式用了几种可疑的 JavaScript 特性，因此存在漏洞：在 [ECMAScript 5严格模式中](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions_and_function_scope/Strict_mode)，是不允许使用 eval 和 with 的，而 with 语法更是被禁止使用的。

但更重要的是，这种实现方式并不能完全满足我们的需求。虽然大部分 Angular 表达式都只是 JavaScript 而已，但也有所扩展：


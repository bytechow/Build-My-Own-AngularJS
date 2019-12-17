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

但更重要的是，这种实现方式并不能完全满足我们的需求。虽然大部分 Angular 表达式都只是 JavaScript 而已，但也有所扩展：过滤器表达式中使用的管道符号 `|` 就不是合法的 JavaScript 字符（这个管道符号实际上是一个位[运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators)，也因此无法通过 `eval` 进行解析)。

同时，允许使用 `eval` 来执行任意 JavaScript 代码，还需要
考虑一些严重的安全问题。当代码来源于 HTML 时尤其如此，也就是允许用户在 HTML 中键入内容。这会给跨站点脚本攻击（cross-site scripting attacks，简称为 XSS）。这也就是为什么 Angular 表达式需要被限定在一个作用域的上下文中执行，而不是像 `window` 这样的全局对象。下面我们也会看到，Angular 也尽可能地防止我们用表达式做任何危险的事情。

那如果表达式不是 JavaScript，那是什么呢？你可以把它称之为一个全新的编程语言。它从 JavaScript 中派生出来，对短表达式进行了优化，去除了大多数的控制流语句，并添加了对过滤器的支持。

在下面章节的开头，我们就开始实现这种编程语言。它会接收一个表达式字符串，然后返回一个可以对表达式内容进行解析计算的函数。我们将进入一个包含语法分析、词汇分析和抽象语法树的新世界，语法分析、词汇分析和抽象语法树其实就是编程语言设计者的工具。

#### 我们会跳过的内容

为了专注于了解表达式的本质，我们会跳过两个 Angular 表达式本身包含的特性：

- 为了在编译出错时显示清晰的错误提示，Angular 表达式做了很多工作。这这涉及到要记忆输入字符串中每个字符和符号的位置。为了保证开发过程更简单清晰，我们会跳过大部分需要记忆位置的内容，代价就是我们会出现一些不大友好的错误提示信息。
- 表达式解析器支持 HTML 的[内容安全策略](https://en.wikipedia.org/wiki/Content_Security_Policy)（CSP，Content Security Policy），当存在解释模式（interpreted mode）就会从默认的编译模式（compiled mode）切换为解释模式。本书只关注编译模式，也就是说我们不支持内容安全策略。

![](/assets/angularjs-expressions.png)
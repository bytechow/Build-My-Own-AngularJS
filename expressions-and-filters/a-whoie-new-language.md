### 一种全新的语言
#### A Whole New Language

那到底什么是 Angular 表达式呢？它们不就是插入到 HTML 标签之中的 JavaScript 代码片段吗？这种说法还不够准确。

Angular 表达式就是为了访问和操作作用域对象上的数据而存在的，除此之外别无它用。表达式本身跟 JavaScript 非常接近，正如 Miško Hevery 指出的，只用几行 JavaScript 代码就可以实现表达式系统的大部分功能：

```js
function parse(expr){
  return function(scope){
    with (scope) {
      return eval(expr);
    }
  }
}
````

这个函数接收一个表达式字符串，然后返回一个函数，这个函数会把字符串解析为 JavaScript 代码然后执行。与此同时，函数用 `with` 语法把这段代码的上下文限定在一个指定的作用域对象内。

这种实现方式是有漏洞的，因为它使用了几种可疑的 JavaScript 特性，因此存在漏洞：我们不赞成使用 eval 和 with，实际上，with 语法已经在[ECMAScript 5严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions_and_function_scope/Strict_mode)中被禁止使用。

但更重要的是，这种实现方式也不能完全满足我们的需求。虽然大部分 Angular 表达式都只是 JavaScript 代码，但也还是有扩展的：过滤器表达式中用到的管道符号 `|` 就不是合法的 JavaScript 字符（管道符号实际上是一个[位运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators)，无法使用 `eval` 进行解析)。

另外，使用 `eval` 来执行任意 JavaScript 代码可能会引发一些严重的安全问题，尤其是在 HTML 中可能嵌入用户生成内容的时候。这会给跨站点脚本攻击（cross-site scripting attacks，简称为 XSS）带来可乘之机。这也就是为什么 Angular 表达式的执行上下文被限定在一个特定的作用域中，而不是像 `window` 这样的全局对象。下面我们会继续看到，Angular 会尽可能地防止我们用表达式做任何危险的事情。

既然表达式不是 JavaScript，它到底是什么呢？我们可以把 Angular 表达式看作一个全新的编程语言。Angular 表达式是一个针对短表达式进行优化的 JavaScript 派生程序，它去除了大部分的控制流语句，并支持通过过滤器进行过滤。

从下一章开始，我们就开始实现这种编程语言，包括：接受一个表达式字符串，然后返回一个可以对表达式内容进行解析计算的函数。这将带我们进入一个有关解析器、词法分析器和抽象语法树的新世界，三者均是编程语言设计者的工具。

#### 我们会跳过的内容

为了更专注于表达式的本质，在实现过程中我们会略过一些 Angular 表达式本身包含的特性：

- 为了在解析出错时能够显示清晰的错误提示，Angular 表达式解析器做了很多工作。这涉及到记忆输入字符串中字符和标记出现的位置。我们会跳过这部分代码，让实现过程更简单，但代价是错误消息对用户不大友好。
- Angular 表达式解析器支持 HTML 的[内容安全策略](https://en.wikipedia.org/wiki/Content_Security_Policy)（CSP，Content Security Policy），若配置了 CSP，解析器就会从默认的编译模式（compiled mode）切换为解释模式。本书只关注编译模式，也就是说我们的代码实现不支持内容安全策略。

![](/assets/angularjs-expressions.png)
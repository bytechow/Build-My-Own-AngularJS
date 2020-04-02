离实现完整的 Angular 表达式语言，还差一个特性——过滤器。

过滤器的作用就是把表达式的结果值转换为其他值。当你对一个表达式应用一个过滤器时，过滤器函数就会把表达式的值作为参数进行调用，而这个函数返回值就会作为这个表达式的最终返回值。

我们可以使用类似 Unix 风格的管道符号来启用一个过滤器。举个例子，Angular 内置的 [uppercase](https://docs.angularjs.org/api/ng/filter/uppercase) 过滤器就会接收一个字符串，然后把字母串中所有字母变成大写：

```
myExpression | uppercase
```

我们也可以将几个过滤器组合起来使用：

```
myNumbers | odd | increment
```

要理解 filter 的关键在于，它们实际上也只是一个普通的函数而已。它们接收一个输入表达式的值，然后输出另一个值，这个值会成为表达式的输出（或者是另一个过滤器的输入）。上面的表达式大致相当于：

```js
increment(odd(myNumbers))
```

过滤器和纯函数之前最主要的区别在于，我们不需要把过滤器放到 Scope 中才能进行调用。相反，你可以在 Angular 应用中注册过滤器，然后就可以在任何需要的地方使用它们。当你使用了管道运算符，Angular 会自动从过滤器注册表（filter registry）中找到对应的过滤器。

跟我们目前看到的所有表达式特性不同的是，过滤器（这个特性）并不存在于 JavaScript 语言中。在 JavaScript 中，管道符号 `|` 意味着一个按位或运算符（bitwise OR operation），但 Angular 用这个符号作为过滤器的标识。

本章，我们将在我们的表达式系统中加入对过滤器的支持。我们还将实现了一个 Angular 内置过滤器：多才多艺、命名很有趣的 `filter` 过滤器。

Angular 本身还实现了[一整套内置过滤器](https://docs.angularjs.org/api/ng/filter)，比如数字过滤器和货币过滤器。我们不会把它们都实现完，但如果你对其中任何一个过滤器感兴趣，它们的[源代码](https://github.com/angular/angular.js/tree/master/src/ng/filter)应该也很好理解。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter8-operator-expressions)
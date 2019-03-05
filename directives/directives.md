到本书这一部分，我们终于可以开始构建被很多人认为是 AngularJS 最核心的特性——指令系统。

在 Angular 的三个特性——digest 循环、依赖注入和指令之中，指令是与构建 Web 浏览器最直接相关的。digest循环和依赖注入这两个特性更多的是作为一个通用特性，而指令则是专门开发出来、用于处理浏览器上的文档对象模型（Docuemnt Object Model，简称DOM）。指令是 Angular “视图层”的最重要组成部分。

指令的概念非常简单，但却也十分强大：一个可扩展的 DOM。除了使用浏览器本身已经实现和标准化的 HTML 元素和属性以外，我们其实还可以自定义一些包含特殊含义和功能的 HTML 元素和属性。可以这样说，有了指令，我们可以在标准 DOM 的基础上，建立一套领域特定语言（Domain-Specific Languages，简称 DSLs）。这套 DSLs 可能专用于某个应用程序，也可能会作为公司内部通用的规范组件，甚至可能被拆分为开源项目。

”可扩展 DOM“这个概念与年代久远的“可扩展编程语言”概念有很多相似点。“可扩展编程语言”概念在`Lisp语言族`中更为普遍。在 Lisp 语言中，不仅仅会用原生的编程语言，还可能会专门为解决某个问题而扩展编程语言，并用这种新的扩展语言来编程。Paul Graham 在他1993年发表的《Programming Bottom-Up》将可扩展编程语言描述为：

> Experienced Lisp programmers divide up their programs differently. As well as top-down design, they follow
a principle which could be called bottom-up design-- changing the language to suit the problem.

> In Lisp, you don’t just write your program down toward the language, you also build the language up toward your program. As you’re writing a program you may think “I wish Lisp had such-and-such an operator.” So you go and write it. Afterward you realize that using the new operator would simplify the design of
another part of the program, and so on.

> Language and program evolve together. Like the border between two warring states, the boundary between
language and program is drawn and redrawn, until eventually it comes to rest along the mountains and
rivers, the natural frontiers of your problem. In the end your program will look as if the language had been
designed for it. And when language and program fit one another well, you end up with code which is clear,
small, and efficient.
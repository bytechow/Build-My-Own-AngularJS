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

如果你已经习惯了编写 Angular 指令，那么你应该不会对上面这段描述感觉陌生。这正正就是我们在 Angular 中会做的：我们会在应用程序中创建 DOM 元素。我们会想：“我希望浏览器拥有像这样那样的元素”。然后我们就会开始尝试创建这个元素。

其实可扩展 DOM 不仅仅是在 Angular 中存在，官方的`Web组件规范（Web Components standard）`现在进入定案和开发阶段。Web 组件很大程度上可以看作是 Angular 指令的进阶版，Web组件在模块化上更进一步，比如，将组件的样式限定为只在组件内部可用。Angular 2将会使用 Web 组件，但 Angular 1.x 版本不会这么做。

指令编译器也是 Angular 源码中最复杂的部分。一是指令系统提供了很丰富的特性，为了实现这些特性，指令做了很多工作，后面我们就会看到了。另外，指令的复杂性还来源于 DOM 标准的繁杂。但最主要的原因还是在编译器本身的复杂性。开发这个编译器需要花费数年，在这期间做了很多各种的添加和修改。如果你正在熟悉指令 API 后，你可能会因为想弄明白指令而抓狂。

这种显而易见的复杂度，让弄明白指令编译器原理这件事情变得更有价值。所以，接下来的几章我们都会讲解指令编译器。在最后，我们会讲解几个实际的指令。
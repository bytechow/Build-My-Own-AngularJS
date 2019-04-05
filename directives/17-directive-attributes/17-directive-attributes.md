要使用指令，处理 DOM 属性是很重要的一部分，无论我们是否使用属性式指令。实际上，元素和样式类指令经常需要与属性“打交道”。就连注释式指令都可能会与属性产生关联，这我们将在本章中看到。

属性可以用于配置指令或向指令传递信息。指令可以通过对属性的操作来改变浏览器中 DOM 元素的外观和表现。除此以外，属性也可以通过一种叫"观察"（observing）的机制，提供指令与指令之间沟通的渠道。

本章我们会开发完整的指令属性功能。学习完本章内容，你就能知道指令中的`compile`和`link`函数接收的`attrs`参数的所有秘密。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter16-dom-compilation-and-basic-directives)
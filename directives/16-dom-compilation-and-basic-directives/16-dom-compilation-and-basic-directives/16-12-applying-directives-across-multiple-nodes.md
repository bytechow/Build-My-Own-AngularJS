### 跨节点应用指令（Applying Directives Across Multiple Nodes）

目前为止，我们看到了指令可以通过4种不同的途径进行匹配。但我们还有一种途径没说，那就是对一连串的兄弟节点应用指令，我们只需要指明要应用范围的开头与结尾即可：

```html
<div my-directive-start>
</div>
<some-other-html></some-other-html>
<div my-directive-end>
</div>
```
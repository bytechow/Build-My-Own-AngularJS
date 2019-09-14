## 利用 JSHint 做静态分析（Enable Static Analysis With JSHint）

JSHint 是一个能够读取 JavaScript 代码然后提供关于代码中出现的语法或结构性错误的分析报告的工具。分析过程被称为_linting_，它对我们十分重要，因为作为一个库的作者，我们并不希望分享一些可能引起问题的代码给他人。

我们可以通过 NPM 获取到 `jshint` 包，那就使用`npm`命令安装这个包下来就可以了：

```bash
npm install --save-dev jshint
```

当你运行这行命令时，会发生几件事：首先一个名为`node_modules`的目录会自动生成在项目内部，而`jshint`包也会下载到这个目录下。

另外，如果你查看一下`package.json`文件，你会看到文件已经发生了改变：现在文件中增加了一个`devDependencies`的 key，而这个 key 对应的对象里面会有一个 `jshint` 的 key。这说明 `jshint` 是我们项目中加入的一个用于开发阶段看的依赖包。造成`package.json`发生如此改变的关键就在于命令中的 `--save-dev` 标识。
### 构建产品包（Building The Production Bundle）

现在我们已经拥有了一个可以进行运行测试的框架了！在我们正式进行测试之前，我们先要把框架打包成一个可以在网页中加载的文件。

我们已经在项目中集成了 Browserify，但目前只是把它用于单元测试。而我们现在缺的是一种能构建产品包的方式。

要解决这个问题也很简单。我们只需要把 bootstrap 文件作为参数，运行 Browserify 就可以了：

```bash
./node_modules/browserify/bin/cmd.js src/bootstrap.js
```